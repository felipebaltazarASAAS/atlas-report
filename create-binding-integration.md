# Integração de bindings no Asaas Mobile

Como consumir, no app, um binding criado no repositório `maui-projects`, seja ele **iOS**, **Android** ou **MAUI** (iOS + Android, no mesmo PR ou em PRs irmãos). Vale para qualquer binding do repositório: os nomes de feature, SDK e operações aparecem como placeholders (`<Feature>`, `<Nome>`, `<Operation>`).

A skill [`create-binding-integration`](../.claude/skills/create-binding-integration/SKILL.md) automatiza este roteiro a partir da URL do PR do binding. Para criar o binding em si, veja a skill `create-binding` e o `docs/iosbinding.md` do `maui-projects`.

## 0. Modos de integração

O modo vem do que o PR do binding contém (`bindings[].platform` na saída do `read_binding_pr.py`):

| Modo | Pacote(s) no PR | O que o app faz |
|---|---|---|
| `ios` | `Asaas.<Nome>.iOS.Maui` | Referência condicional `-ios` e `Platforms/iOS` (seções 4 e 5). Se o módulo já tem Android, espelha o contrato dele |
| `android` | `Asaas.<Nome>.Android.Maui` | Referência condicional `-android` e `Platforms/Android` (seções 4 e 5B). Se o módulo já tem iOS, espelha o contrato dele |
| `maui` | iOS e Android juntos, ou o pacote cross `Asaas.<Nome>.Maui` | As duas plataformas, cada uma no seu `Platforms/`; o pacote cross entra sem condição de plataforma (seção 4) |

Nos três modos o contrato `I<Feature>` e o `Shared/` são únicos. A plataforma que **já existe** no módulo é a referência de contrato para a nova; quando o módulo é novo e as duas plataformas chegam juntas, o `I<Feature>` é definido a partir do que o produto precisa (seção 6) e as duas implementações o cumprem.

## 1. Fluxo de ponta a ponta

1. O binding é criado e mergeado no `maui-projects` (pacote `Asaas.<Nome>.iOS.Maui`, `Asaas.<Nome>.Android.Maui` ou `Asaas.<Nome>.Maui`).
2. O CI publica o pacote no feed `github` (`nuget.config`, padrão `Asaas.*`): versão estável e previews `-pre<N>`.
3. O app referencia o pacote só na plataforma dele (ou sem condição, no cross) e implementa o contrato da feature em `Platforms/<plataforma>`.
4. O contrato `I<Feature>` e as classes de `Shared/` continuam únicos para as duas plataformas.

O pacote precisa existir no feed antes de qualquer trabalho no app. O script `.claude/skills/create-binding-integration/scripts/read_binding_pr.py <url-do-pr>` lê o PR do binding (iOS, Android ou cross) e devolve em JSON um item por plataforma, com o pacote, o contrato e o estado no feed. Para conferir só o feed (troque `ios` por `android` ou remova o segmento no cross):

```bash
curl -s -u "asaasdev:$NUGET_GITHUB_TOKEN" https://nuget.pkg.github.com/asaasdev/download/asaas.<nome>.ios.maui/index.json
```

## 2. Estrutura do módulo

```
App/Shared/Asaas.<Feature>/
├── Asaas.<Feature>.csproj
├── Shared/                         ← único para as duas plataformas
│   ├── Interfaces/I<Feature>.cs    ← contrato (Task, sem tipos do SDK)
│   ├── Managers/<Feature>Manager.cs
│   ├── Dtos/                       ← *DTO
│   ├── Enums/
│   ├── Helpers/                    ← constantes e parsers sem dependência de SDK
│   ├── Trackers/
│   └── <Feature>*Wrapper / *Response / *Data
└── Platforms/
    ├── Android/
    │   ├── <Feature>Impl.cs        ← um método por chamada ao SDK, com TaskCompletionSource
    │   ├── <Feature>*ListenerImpl.cs ← um listener por interface Java do SDK
    │   └── <Feature>*Extensions.cs ← conversões de enums do SDK
    └── iOS/
        ├── <Feature>Impl.cs
        ├── <Feature>Service.cs     ← mapeamentos SDK → tipos do app
        └── Handlers/               ← um handler por integração com o SDK
```

Namespaces: `Asaas.<Feature>.Shared[.X]` para `Shared/`, `Asaas.<Feature>` para as implementações de plataforma (e listeners do Android) e `Asaas.<Feature>.Handlers` para os handlers do iOS.

## 3. O que vai para `Shared/` e o que fica na plataforma

Critério: lógica que não referencia tipo do SDK nativo vai para `Shared/`; o resto fica na plataforma. Isso é feito antes de escrever a nova plataforma, e é verificado compilando as duas plataformas. Vale nos dois sentidos: ao integrar iOS, o que está em `Platforms/Android` sem tipo de SDK sobe para `Shared/`; ao integrar Android em módulo que já tem iOS, o mesmo vale para `Platforms/iOS`. Em módulo novo, o que for independente de SDK já nasce em `Shared/`.

| Vai para `Shared/` | Fica em `Platforms/<plataforma>` |
|---|---|
| DTOs de JSON do backend (sufixo `DTO`, uma classe por arquivo) | Listeners e interfaces do SDK (no Android, interfaces Java; no iOS, handlers) |
| Constantes sem SDK (`Helpers`) | Extensions e conversões de enums do SDK |
| Parsers de strings e datas que o SDK devolve, iguais nas duas plataformas | Conversões de unidade que dependem do formato do SDK (por exemplo, centavos contra string decimal) |
| Contrato `I<Feature>`, wrappers, responses e `*Data` | Provedores de texto ou recursos do SDK |

Não criar interfaces que espelhem o SDK nativo. O contrato compartilhado é `I<Feature>`; os contratos dos SDKs são diferentes demais (listeners Java contra blocos Obj-C) e uma interface comum acabaria sendo o próprio `I<Feature>`.

## 4. Referência do pacote

No `Asaas.<Feature>.csproj`, a referência é condicional à plataforma do pacote, exceto no pacote cross, que entra sem condição. Use o bloco do modo escolhido; no `maui` combine os dois, um por plataforma.

### 4.1 Padrão: versão fixa

```xml
<ItemGroup Condition="$(TargetFramework.EndsWith('-ios'))">
    <PackageReference Include="Asaas.<Nome>.iOS.Maui" Version="1.0.0"/>
</ItemGroup>

<ItemGroup Condition="$(TargetFramework.EndsWith('-android'))">
    <PackageReference Include="Asaas.<Nome>.Android.Maui" Version="1.0.0"/>
</ItemGroup>
```

É a forma de `Asaas.Zendesk` e `Asaas.GoogleWallet`. Dependências que o `nuspec` do binding Android declara e o app ainda não referencia (por exemplo, pacotes `Xamarin.AndroidX.*` ou `Square.*`) entram no mesmo `ItemGroup` condicional, com a versão que o `nuspec` pede.

**Cross (`Asaas.<Nome>.Maui`)**: pacote único para as duas plataformas, referenciado sem condição de `TargetFramework`, como `Asaas.BarcodeScanning.Native.Maui`.

### 4.2 Opcional: versão em variável, com preview em Debug

**Motivo:** o `Asaas.TapOnPhone` precisa da versão do SDK da Symbiotic como variável do MSBuild para gerar os tracks (a versão é gravada em `AssemblyMetadataAttribute` e enviada nos eventos). Só use este formato quando a feature precisar ler a versão do SDK, ou quando o time quiser consumir previews em Debug. Sem essa necessidade, fique no 4.1.

```xml
<PropertyGroup>
    <<Nome>IosSdkVersion>1.0.0</<Nome>IosSdkVersion>
</PropertyGroup>

<Choose>
    <When Condition="$(TargetFramework.EndsWith('-ios')) AND '$(Configuration)' == 'Debug'">
        <ItemGroup>
            <PackageReference Include="Asaas.<Nome>.iOS.Maui" Version="$(<Nome>IosSdkVersion)-pre*"/>
        </ItemGroup>
    </When>
    <When Condition="$(TargetFramework.EndsWith('-ios')) AND '$(Configuration)' != 'Debug'">
        <ItemGroup>
            <PackageReference Include="Asaas.<Nome>.iOS.Maui" Version="$(<Nome>IosSdkVersion)"/>
        </ItemGroup>
    </When>
</Choose>

<ItemGroup Condition="$(TargetFramework.EndsWith('-ios'))">
    <AssemblyAttribute Include="System.Reflection.AssemblyMetadataAttribute">
        <_Parameter1><Nome>SdkVersion</_Parameter1>
        <_Parameter2>$(<Nome>IosSdkVersion)</_Parameter2>
    </AssemblyAttribute>
</ItemGroup>
```

Para o Android, troque `-ios` por `-android`, `iOS` por `Android` e `IosSdkVersion` por `AndroidSdkVersion` (modelo em `Asaas.TapOnPhone`). O `AssemblyAttribute` só entra quando a versão for usada nos tracks. Debug aceita preview; Release fixa a estável. O intervalo `-pre*` também resolve a versão estável quando não há preview mais novo.

O `.csproj` costuma ter BOM: preserve.

## 5. Padrão da implementação iOS (5B para Android)

No iOS os callbacks já são `Action`, então o equivalente do listener do Android é o **handler**.

### 5.1 Regras

1. **Uma chamada ao SDK por método privado**, cada uma com o seu próprio try-catch.
2. **O handler nasce dentro do método** que faz a chamada. Quem compõe (o método público) não cria handler.
3. **Nenhuma chamada ao SDK dispara outra** dentro de um callback. A composição é feita no método público, com `await`.
4. **Toda a lógica de reação ao SDK fica no handler**: tracker, log, status e montagem da resposta.
5. **O handler tem sempre as mesmas partes**: um `TaskCompletionSource`, `GetResult()`, `OnSuccess`, `OnFailure(NSError)` e `OnUnexpectedError(Exception)`. Operações com progresso também expõem `OnEvent`.
6. **Mapeamentos puros** (formato do valor, enums do SDK → enums do app, SDK → `*Data`) ficam em `<Feature>Service`, estático, em `Platforms/iOS`.
7. **Erro do SDK** vira o resultado de falha do contrato, com o código (`error.Code.ToString()`); **exceção inesperada** registra com `AsaasLogger.Error` e completa com o valor neutro do tipo (`false`, `null` ou uma resposta de falha sem código), de modo que o chamador não receba `null` quando o contrato não permite.
8. **Callbacks de carregamento ou progresso do Android:** se o fluxo do app depende de um callback que o listener Android dispara (por exemplo, "carregamento concluído"), o handler iOS dispara o equivalente, uma única vez e também ao final da operação, inclusive nos caminhos de falha. Sem isso o fluxo que espera o callback não prossegue.
9. Sem comentários e sem `var`, como no resto do projeto (`CLAUDE.md`).
10. **Inicialização do SDK:** se as demais operações dependem de `Initialize`, o `Impl` guarda a `Task<bool>` da inicialização e cada operação dependente a aguarda antes de chamar o SDK (`EnsureInitialized`). Se a inicialização anterior terminou com falha, ela é refeita. O resultado do `Initialize` nunca é descartado: sem isso, uma operação chamada logo depois pode rodar antes de o SDK estar pronto e falhar sem diagnóstico.
11. **Conversão de identificadores:** valores que vieram do SDK ou do backend (como o identificador da transação guardado para estorno) são convertidos com `TryParse`; se a conversão falhar, o método devolve a falha do contrato.

### 5.2 Handler

```csharp
using Asaas.Measurement.Shared.Logger;
using Foundation;

namespace Asaas.<Feature>.Handlers;

public class <Feature><Operation>Handler {

    private readonly TaskCompletionSource<<TResult>> _taskCompletionSource = new();

    public Task<<TResult>> GetResult() => _taskCompletionSource.Task;

    public void OnSuccess(<SdkResult> result) {
        <Feature>Tracker.<Operation>Succeeded();
        _taskCompletionSource.TrySetResult(<successValue>);
    }

    public void OnFailure(NSError error) {
        AsaasLogger.Warn($"<PREFIXO> - <Feature><Operation>Handler.OnFailure >> <mensagem> - Code: {error.Code} - {error.LocalizedDescription}");
        <Feature>Tracker.<Operation>Failed(error.Code.ToString());
        _taskCompletionSource.TrySetResult(<failureValue>);
    }

    public void OnUnexpectedError(Exception exception) {
        AsaasLogger.Error(exception, "<PREFIXO>");
        _taskCompletionSource.TrySetResult(<failureValue>);
    }
}
```

`<TResult>` é o que o método devolve ao chamador (`bool`, a resposta do contrato, um tipo do SDK). Quando o SDK devolve o resultado sem argumento (`Action`), `OnSuccess` não recebe parâmetro.

### 5.3 Método de chamada ao SDK

```csharp
private static Task<<TResult>> <Operation>(<parâmetros>) {
    <Feature><Operation>Handler handler = new();

    try {
        <Nome>ProxySDK.<Operation>(<argumentos>, handler.OnSuccess, handler.OnFailure);
    } catch (Exception e) {
        handler.OnUnexpectedError(e);
    }

    return handler.GetResult();
}
```

### 5.4 Composição no método público

Operações que dependem de outra (por exemplo, buscar um dado e depois usá-lo) são encadeadas por `await` no método público, e a falha do primeiro passo é convertida no resultado de falha do contrato ali, sem que um handler conheça o outro:

```csharp
public async Task<<ContractResult>> <PublicOperation>(<parâmetros>) {
    <SdkData>? data = await <FirstOperation>(<parâmetros>);

    if (data is null) return Build<FirstOperation>FailedResponse(<parâmetros>);

    return await <SecondOperation>(data, <parâmetros>);
}
```

## 5B. Padrão da implementação Android

O SDK Android reage por interfaces Java. Cada interface vira uma classe de listener em `Platforms/Android`, e cada operação vira um método do `Impl` com `TaskCompletionSource` e try-catch próprio:

```csharp
public class <Feature><Operation>ListenerImpl(Action onSuccess) : Java.Lang.Object, I<Operation>Listener {

    public void OnSuccess() {
        onSuccess();
    }
}
```

Regras:

1. Um listener por interface Java do SDK, com o `Impl` do módulo como único consumidor; construtor primário com as ações que ele dispara.
2. Uma chamada ao SDK por método, cada uma com o seu `TaskCompletionSource` e try-catch; o método público compõe com `await`.
3. Erro do SDK vira o resultado de falha do contrato; exceção inesperada registra com `AsaasLogger.Error` e completa com o valor neutro do tipo.
4. Conversões de enums do SDK ficam em `*Extensions` na plataforma; strings e recursos do SDK, em um provider (`<Feature>StringProvider`).
5. Os tipos Java (`Java.Lang.Object`, `Com.<Fornecedor>...`) nunca vazam para `Shared/`.
6. Sem comentários e sem `var` (`CLAUDE.md`).
7. Os nós removidos do binding (`android.removedNodes`) e as coordenadas Maven (`android.libraries`) mostram o que o proxy deixou de fora e a versão do SDK nativo; confira ambos no relatório.

Em módulo que já tem iOS, o contrato `I<Feature>` e os `*Data` existentes definem o que o Android devolve; a diferença de formato entre as plataformas (valores, datas, status) é resolvida em `Platforms/Android`, não em `Shared/`.

## 6. Contrato do SDK e decisões de mapeamento

1. **Liste as operações que o SDK oferece e escolha o escopo.** A lista vem do contrato do binding (`ApiDefinitions.cs`): cada método com assinatura, seletor e se é estático. O script `read_binding_pr.py <url-do-pr> --operations` imprime só isso, por interface; modelos aparecem com a contagem de propriedades. Mostre a lista ao responsável pela feature, marcando as que espelham um método do contrato `I<Feature>`, e pergunte **uma operação por vez** se ela entra ou não, em vez de agrupar várias numa só pergunta: assim cada operação é decidida de forma explícita e nenhuma entra por estar no mesmo grupo de outra. O SDK costuma oferecer mais do que a feature usa (histórico, comprovantes, onboarding, ajuda do fornecedor): só entra o que foi escolhido, e o restante fica registrado como fora do escopo. Operações que as escolhidas exigem (inicialização, ativação) entram junto, avisando quem escolheu.
2. **Opcional (quando houver binding ou implementação da outra plataforma):** monte a tabela Android × iOS com uma linha por método do contrato `I<Feature>`: como o Android faz, como o binding iOS faz (assinatura do `ApiDefinitions.cs`) e o que não tem equivalente. No modo `android`, a coluna do iOS é a implementação existente, se houver; no `maui` com módulo novo, a tabela parte do `I<Feature>` desejado.
3. **Documentação do fornecedor (6A).** O que o `ApiDefinitions.cs` não diz vem do guia do fornecedor; leia-o antes de decidir mapeamentos.
4. Quando houver plataforma de referência, espelhe a que já existe no módulo por padrão (no iOS, o Android; no Android, o iOS). Onde não houver equivalente claro, pergunte ao responsável pela feature em vez de decidir sozinho. Costumam precisar de confirmação:
   - mapeamento de ambientes (teste e produção) para os valores do SDK;
   - operações com semântica parecida e consequências diferentes (por exemplo, cancelar contra devolver);
   - mapeamento de tipos de operação, de resultados e de status para os enums do app;
   - formato de valores monetários, datas e identificadores que o SDK recebe e devolve;
   - quais dados do objeto original o SDK exige para operações que dependem de uma operação anterior.
5. Dados que o SDK devolve e o app precisa guardar para operações futuras devem ser guardados exatamente como vieram. Não reconstruir objetos do SDK com campos vazios: siga a documentação do fornecedor sobre como recuperar o original. Antes de definir o que o app guarda, rastreie quem lê o campo e confirme tipo e semântica. A operação futura recupera o original pelo meio oficial do SDK (por exemplo, consulta por identificador), não por cache em memória, que se perde ao reiniciar o app.
6. **Strings em formato não fixado pelo fornecedor** (datas, horas, valores): não assuma um formato. O parser tenta uma lista de formatos conhecidos (incluindo o do histórico, quando for diferente do da operação), resolve a ausência de ano sem produzir data no futuro (se a data montada for futura, usa o ano anterior) e, se nenhum formato servir, devolve o fallback definido com a feature e registra um warning com o valor cru. O parser fica em `Shared/Helpers` quando as plataformas compartilham formatos.
7. **Enums e erros do SDK:** todos os valores de cada enum de resultado têm destino explícito no mapeamento, e os códigos de erro relevantes têm status próprio no app (cancelado, tempo esgotado, sem suporte). O padrão `_ =>` só cobre o que o usuário confirmou como falha genérica.
8. Em ambientes de teste, qualquer leitor, stub ou modo simulado do SDK só pode ser usado em `DEBUG` e em dispositivo virtual. Nunca em Release.

## 6A. Documentação do fornecedor

O PR do binding traz o contrato (assinaturas, enums, modelos); o guia de integração do fornecedor traz o comportamento. Antes de fechar o plano, extraia dele, com o `.swiftinterface` ou o `.xcframework` como conferência de assinatura (iOS):

| Item | Para que serve |
|---|---|
| Formato de cada string devolvida | Decide o parser tolerante (seção 6, item 6) |
| Tabela de códigos de erro | Define os status do app por código (seção 6, item 7) |
| Valores completos dos enums de resultado | Mapeamento exaustivo (seção 6, item 7) |
| Requisitos de SO e de dispositivo | Verificação de suporte do dispositivo na `Impl` |
| Ordem das chamadas e thread dos callbacks | Regra 10 da seção 5.1 e `MainThread` onde o SDK chegar em background |
| Recuperação de dados de operações anteriores | Estorno e consultas (seção 6, item 5) |

O que o guia não fixa fica registrado como "não documentado" e vai para o relatório e para a seção 7.

## 7. Pontos em aberto com o fornecedor

Registre no relatório o que o proxy do binding não cobre ou o que o SDK deixa a critério da integração. Costumam aparecer:

- parâmetros que o SDK aceita e o proxy não expõe;
- pré-requisitos de onboarding que o app ainda não chama;
- cancelamento, timeout e outros desfechos que numa plataforma têm callback próprio e na outra chegam como erro (hoje, no Android há callback e no iOS chega como erro);
- formatos de data, hora e valor devolvidos como string (use o parser tolerante da seção 6, item 6);
- diferenças de comportamento para respostas nulas ou parciais entre as plataformas.

## 8. Verificação

Compilar **os dois alvos, qualquer que seja o modo**, e exigir 0 erros: a plataforma não integrada também precisa continuar compilando (ela pode ter recebido código movido para `Shared/`).

```bash
dotnet build App/Shared/Asaas.<Feature>/Asaas.<Feature>.csproj -f net10.0-ios -c Debug
dotnet build App/Shared/Asaas.<Feature>/Asaas.<Feature>.csproj -f net10.0-android -c Debug
dotnet build App/AsaasMobile.Core/AsaasMobile.Core.csproj -f net10.0-ios -c Debug
dotnet build App/AsaasMobile.Core/AsaasMobile.Core.csproj -f net10.0-android -c Debug
```

Compilar não prova que os seletores (iOS) ou as classes Java (Android) do binding funcionam em tempo de execução. O teste real é no simulador ou no aparelho, e deve ser dito como não feito quando não for.

Os projetos de feature não têm testes unitários; não declarar a integração como verificada só pela compilação.
