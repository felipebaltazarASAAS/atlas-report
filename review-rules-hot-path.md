# Regras de Review — Hot Path, Startup e Handlers Globais

> Referência de código para revisor: este documento é um guia rápido das regras aplicadas automaticamente em TODAS as reviews de código. Para detalhes completos, exemplos e contexto, consulte [`.claude/rules/review-hot-path-en.md`](../.claude/rules/review-hot-path-en.md).

## Contexto

Estas regras foram criadas após o incidente de 2026-04-23 ([postmortem](../post-mortem-engenharia/docs/general/postmortem-app-asaas-2026-04-23_instabilidade_home_loop_serilogger.md)), quando alterações em hot paths, startup e handlers globais causaram ANR (Application Not Responding) e instabilidade na Home:

1. **Recriação desnecessária de Serilogger** a cada log → consumo excessivo de CPU/memória
2. **Loop infinito em `FirstChanceException` + I/O bloqueante** → ANR na thread principal

O custo dessas alterações só aparece em volume/tempo de execução, não em testes manuais — por isso estas regras se aplicam a **TODAS** as reviews de código.

---

## Regras

### HOT-001 — Alocação e Inicialização em Hot Path

**Contexto**: Métodos chamados a cada log, request, evento, binding ou render.

**A regra**: Ao revisar alterações em código hot path, sempre responder:
> "Isso aloca/inicializa algo a cada chamada?"

Se sim, comentar exigindo que a criação ocorra uma única vez (campo `static`, singleton ou `Lazy<T>`).

**Detecta**:
- Criação de loggers (`CreateLogger()`, `BuildLogger()`, configuração de sinks)
- Instanciação de `HttpClient`, serializers, `Regex` sem `RegexOptions.Compiled` cacheado
- Leitura de configuração dentro de método hot path

**Causa do incidente**: PR #1371 recriava o Serilogger a cada chamada de `DatadogLogger.Log()`.

**Exemplo**:

❌ **Incorreto**:
```csharp
public void Log(string message) {
    LoggerConfiguration loggerConfiguration = BuildLogger();
    Logger logger = loggerConfiguration.CreateLogger();
    logger.Information(message);
}
```

✅ **Correto**:
```csharp
private static readonly Logger _logger = BuildLogger().CreateLogger();

public void Log(string message) {
    _logger.Information(message);
}
```

---

### HOT-002 — Handlers Globais de Exceção

⚠️ **CRÍTICO** — Esta regra previne o cenário exato do incidente de 2026-04-23.

**Contexto**: Subscrição em `AppDomain.CurrentDomain.FirstChanceException`, `UnhandledException`, `TaskScheduler.UnobservedTaskException` ou equivalentes.

**Regra 2.1 — Guard de Reentrância**:
Todo handler global deve ter guard de reentrância (flag por thread que faz o handler retornar imediatamente se já estiver tratando uma exceção). Sem o guard, uma exceção lançada dentro do handler gera loop infinito.

**Regra 2.2 — Proibido I/O Bloqueante**:
Proibido dentro de handler global:
- `Preferences.Set`/`Get`
- Escrita em arquivo
- Chamada de rede síncrona
- Qualquer operação que adquira lock

No incidente de 2026-04-23, `Preferences.Set` dentro de `FirstChanceException` bloqueou a main thread em `pthread_cond_wait()` e causou ANR com app force-closed.

**Regra 2.3 — Rate Limiting e Fail-Silent**:
- Rate limiting: máximo de N eventos processados por sessão
- Fail-silent: corpo inteiro protegido em try/catch, nunca propagar exceção

**Regra 2.4 — FirstChanceException é Caríssimo**:
`FirstChanceException` dispara para **toda** exceção do processo, inclusive as já tratadas — qualquer custo no handler é multiplicado pelo volume total de exceções do app. Uso dessa API deve ser questionado no review.

**Exemplo**:

❌ **Incorreto** (gatilho direto do ANR):
```csharp
AppDomain.CurrentDomain.FirstChanceException += (sender, args) => {
    Preferences.Set("LastException", args.Exception.ToString());
};
```

✅ **Correto**:
```csharp
[ThreadStatic]
private static bool _isHandlingException;
private static int _handledExceptionCount;
private const int MAX_EXCEPTIONS_PER_SESSION = 100;

private static void OnFirstChanceException(object sender, FirstChanceExceptionEventArgs args) {
    if (_isHandlingException) return;
    if (Interlocked.Increment(ref _handledExceptionCount) > MAX_EXCEPTIONS_PER_SESSION) return;

    _isHandlingException = true;

    try {
        ExceptionBuffer.EnqueueForAsyncPersistence(args.Exception);
    } catch {
        // Fail-silent
    } finally {
        _isHandlingException = false;
    }
}
```

---

### HOT-003 — Código de Inicialização (Startup)

**Contexto**: Alterações em `App.cs`, `MauiProgram.cs`, `AsaasServices.cs`, `FrameworkApplication`, `InitialLoadPage*` e construtores de classes instanciadas no boot.

**Regra 3.1 — Sem I/O Bloqueante na Main Thread**:
Proibido adicionar I/O bloqueante durante a inicialização:
- Rede síncrona
- Leitura/escrita de arquivo
- `Preferences` em volume

Operações de startup devem ser assíncronas ou adiadas para depois do primeiro frame.

**Regra 3.2 — Cuidado com Preferences no Boot**:
`Preferences`/SharedPreferences é contencioso durante o boot do Android: múltiplos acessos concorrentes aguardam o mesmo lock. Acessos em série no startup devem ser sinalizados.

**Regra 3.3 — Impact no Cold Start**:
Alterações que adicionem trabalho ao caminho de inicialização devem ser questionadas no review quanto ao impacto no tempo de cold start (referência: teste `tests/StartupUiTest`).

---

### HOT-004 — Instrumentação Temporária

**Contexto**: Trackers e instrumentação adicionados para investigação de problemas (ex.: `HomeIssueTracker`, `FirstChanceExceptionTracker`).

**Regra 4.1 — Gateada por Remote Config**:
Instrumentação temporária deve ser gateada por Firebase Remote Config (`FirebaseRemoteConfigManager`), desligada por padrão e desativável sem novo release na loja.

**Regra 4.2 — Mesmo Rigor que Hot Path**:
Código de instrumentação segue as mesmas regras de hot path e handlers (HOT-001 e HOT-002) — o rigor do review deve ser **maior**, não menor, por rodar em caminhos quentes e em estados anômalos do app.

**Regra 4.3 — Sem Deactivation Mechanism = Violação**:
Instrumentação sem mecanismo de desativação remota deve ser sinalizada como violação, citando o custo de remoção (build + publicação + propagação nas lojas, ciclo de dias).

---

## Como Revisar

### Checklist Rápido

- [ ] **Hot Path**: Há alocação ou inicialização dentro de loop/callback/request handler? → HOT-001
- [ ] **FirstChanceException ou handlers globais**: → HOT-002 (guard, sem I/O bloqueante, rate limit, fail-silent)
- [ ] **App.cs / MauiProgram.cs / AsaasServices.cs**: Há I/O bloqueante no construtor ou inicialização? → HOT-003
- [ ] **Instrumentação nova (trackers, logs extra)**: Gateada por Remote Config? Sem forma de desligar = violação → HOT-004

### Onde Encontrar Hot Paths no Código

- `DatadogLogger.cs`, `Serilog` wrappers
- `RequestManager` implementations (API calls)
- `Tracker.*` (analytics)
- Converters (`IValueConverter`)
- Bindings e data templates
- `OnPropertyChanged` em ViewModels
- Event handlers globais

---

## Referências

- 📄 **Documentação completa com exemplos**: [`.claude/rules/review-hot-path-en.md`](../.claude/rules/review-hot-path-en.md)
- 📄 **Postmortem do incidente**: [Instabilidade na Home (2026-04-23)](../post-mortem-engenharia/docs/general/postmortem-app-asaas-2026-04-23_instabilidade_home_loop_serilogger.md)
- 🎯 **Tarefa de implementação**: [MBT-4280](https://asaasdev.atlassian.net/browse/MBT-4280) — Melhorar regras do AmazonQ relacionado a testes alterações de grande impacto
