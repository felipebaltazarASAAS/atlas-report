# Crash no boot em Android 32-bit — `aot-runtime.c ... condition 'plt_entry' not met`

> Guia de diagnóstico para um crash de inicialização que **não** tem causa no código do PR. Se você chegou aqui vindo de um `git bisect`, pare o bisect e leia a seção [Como diagnosticar em 2 minutos](#como-diagnosticar-em-2-minutos).

> **Status — 2026-08-24.** A causa foi eliminada em 2026-08-18: o código do app saiu do assembly principal para a class library `AsaasMobile.Core`, e a maior imagem AOT de 32 bits caiu de 42,1 MiB para 2,6 MiB (ver [Correção aplicada](#correção-aplicada) e `docs/asaas-mobile-core-migration.md`). O runbook segue valendo para uma reincidência, que continua possível: a correção depende de um default do SDK que **nada no repositório fixa**. O guard roda automaticamente em todo build de Release Android, pelo `build-files/android/AotTextSizeGuard.targets` (ver [Prevenção](#prevenção)).
>
> Em 2026-08-24 o mecanismo foi **demonstrado**, não mais inferido: desmontagem dos binários, código-fonte do runtime e A/B no aparelho do incidente (ver [Desmontagem](#desmontagem-o-mecanismo-observado), [O lado do runtime](#o-lado-do-runtime-o-que-o-código-faz) e [Experimento controlado](#experimento-controlado-2026-08-24)).
>
> Esta versão incorpora as medições de 2026-08-13 a 2026-08-18. Três afirmações da versão anterior foram **corrigidas**, não complementadas: o limiar de falha deste app não foi 32 MiB, profile de AOT customizado **não** resolve o assembly principal, e `.text` abaixo de 32 MiB não é margem segura conhecida.

## Sintoma

App Release aborta na inicialização, **somente em devices ARM 32-bit** (`armeabi-v7a`), com um assert nativo do runtime Mono:

```
F monodroid: * Assertion at /__w/1/s/src/mono/mono/mini/aot-runtime.c:5339, condition `plt_entry' not met
F monodroid: Abort at mono-log-adapter.cc:46:3 (...MonodroidRuntime::mono_log_handler...)
--------- beginning of crash
F libc    : Fatal signal 6 (SIGABRT), code -6 in tid 1153 (asaas.asaas)
```

Características que identificam este caso:

- Não é exception gerenciada — não há stack trace .NET, é `SIGABRT` do runtime nativo.
- Acontece **antes** de qualquer código de feature rodar (crash no boot).
- Reproduz em `armeabi-v7a` e **não** reproduz em `arm64-v8a` — confirmado em 2026-08-13, com a mesma imagem AOT de 64 bits (41.128.624 bytes) subindo normalmente. O alcance do branch em A64 é 4× maior.
- Aparece "a partir de um commit" cujo diff não tem nada relacionado a startup, DI, XAML de App ou runtime.
- É determinístico: no build afetado, **todo** boot em 32 bits falha e **nenhum** boot em 64 bits falha. Não há contorno pelo lado do usuário (reinstalar ou limpar dados não muda nada).

## Causa raiz

A PLT de uma imagem AOT ficou a mais de 32 MiB do início do próprio `.text`, além do alcance do branch direto do ARM 32-bit.

O build Release do Android usa AOT (veja `obj/Release/net*.0-android/build.props`: `aotassemblies=true`, `androidaotmode=normal`, `androidenableprofiledaot=true`, `androidlinkmode=sdkonly`). Cada assembly gera uma biblioteca nativa `lib/<abi>/libaot-<Assembly>.dll.so`, contendo o código nativo dos métodos **e** a PLT do módulo, na mesma seção `.text`.

Limites de alcance de branch:

| Arquitetura | Instrução | Alcance |
|---|---|---|
| ARM 32-bit (A32) | `BL` | ±32 MiB |
| ARM 32-bit (Thumb-2) | `BL` | ±16 MiB |
| ARM 64-bit (A64) | `BL` | ±128 MiB |

Dentro dessa `.so`, o Mono emite a PLT **depois** do código dos métodos. Quando a área de métodos cresce além de 32 MiB, os call sites do começo do `.text` deixam de alcançar a PLT com um `BL` direto, e o linker (lld) insere *range-extension thunks* (veneers) no início do `.text`. A partir daí, esses call sites apontam para o veneer, não para a entrada da PLT.

O runtime, no caminho de resolução de PLT (`aot-runtime.c`), **decodifica a instrução do call site** para recuperar o endereço da entrada da PLT que o chamou. Com o veneer no caminho, o endereço obtido não cai dentro de `[plt, plt_end]`, o resultado é `NULL`, e o assert `plt_entry` falha → `SIGABRT`.

O crash cai no boot quando uma das chamadas afetadas é executada na inicialização — o que depende do layout, não só do tamanho.

> **Nível de certeza.** Em 2026-08-24 a cadeia foi fechada dos dois lados: o lado do linker por desmontagem dos binários ([Desmontagem](#desmontagem-o-mecanismo-observado)) e o lado do runtime pelo código-fonte do .NET 9 ([O lado do runtime](#o-lado-do-runtime-o-que-o-código-faz)). O comportamento dos dois builds no aparelho também deixou de ser dedução: foram instalados lado a lado no SM-J510MN ([Experimento controlado](#experimento-controlado-2026-08-24)). Continua **não observado** apenas qual método do boot dispara o assert — os `.so` são stripped, não há como nomeá-lo.

### Dois números diferentes: 32 MiB e ~41,8 MiB

Não confunda os dois:

- **32 MiB** é o teto físico do alcance do `BL` em A32.
- **~41,8 MiB** foi o tamanho de `.text` com que *este app* quebrou. Com 41,74 MiB ele subia; com 41,79 MiB abortava. A diferença entre os dois builds é de **+42 KB**.

A desmontagem mostrou que **nenhum dos dois números é o que governa a falha**. O que governa é a distância entre o call site e a PLT — e a PLT do Mono não fica no começo do arquivo, fica **depois de todo o código dos métodos**, a +32,07 MiB (build bom) e +32,10 MiB (build que quebrou) do início do `.text`.

Daí decorre a regra real:

> A falha se torna possível quando **a área de métodos que antecede a PLT passa de 32 MiB** — porque a partir daí o começo do `.text` deixa de alcançar a PLT com um `BL` direto. O tamanho total do `.text` só importa porque empurra a PLT para frente.

E daí decorre também por que o app conviveu com isso: **os dois builds já estavam nesse regime** — os dois têm veneers apontando para a PLT. O que separa subir de abortar não é cruzar o limite, é *quantos* call sites ficaram do lado de fora e se algum deles roda no boot: 132 contra 391.

Isso foi verificado no aparelho, não deduzido: em 2026-08-24 os dois APKs assinados foram instalados em sequência no mesmo SM-J510MN e o de 41,74 MiB subiu normalmente, com suas 132 chamadas já passando por veneer (ver [Experimento controlado](#experimento-controlado-2026-08-24)).

**Consequência prática:** o tamanho em que a falha aparece **não é um número fixo**; muda com o layout de cada build. Não trate nenhum valor entre 32 MiB e ~42 MiB como seguro, e não conclua que um build está são porque um build anterior de tamanho parecido subiu — o de 41,74 MiB já tinha 132 chamadas passando por veneer.

### Desmontagem: o mecanismo observado

Feita em 2026-08-24 sobre os dois `AsaasMobile.dll.so` de `armeabi-v7a` preservados da investigação original (`llvm-objdump`/`llvm-readelf` do NDK 29). Os binários são stripped, então tudo abaixo veio de reconhecimento de padrão de instrução, não de símbolos.

**A PLT do Mono** — entradas de 16 bytes, indiretas via GOT, localizadas depois do código dos métodos:

```
209f040: e59fc000     ldr  r12, [pc]        ; offset do slot no GOT
209f044: e79ff00c     ldr  pc, [pc, r12]    ; salto indireto
209f048: 00a90720     .word <got offset>
209f04c: 001ed0cc     .word <info>
```

**Os veneers do lld** — entradas de 12 bytes, no primeiro byte do `.text`, saltando por PC-relativo:

```
85b60: e59fc000     ldr  r12, [pc]
85b64: e08ff00c     add  pc, pc, r12       ; salto longo, PC-relativo
85b68: <delta>      .word
```

Medições nos dois binários:

| | `92cb32ab` (subia) | `4bdcb74f` (abortava) |
|---|---:|---:|
| `.text` | 41,74 MiB | 41,79 MiB |
| PLT: posição dentro do `.text` | +32,07 MiB | +32,10 MiB |
| PLT: entradas | 29.644 | 29.668 |
| Veneers no início do `.text` | **132** | **391** |
| Veneers cujo destino é uma entrada da PLT | 132 de 132 (100%) | 391 de 391 (100%) |
| Distância veneer → PLT | 32,07 MiB | 32,10 MiB |
| Maior `BL` da imagem | — | **32,00 MiB** (o máximo codificável) |

Três coisas que essa tabela prova, e que antes eram inferência:

1. **Os veneers existem** e estão exatamente onde a teoria previa: no começo do `.text`, alcançáveis por `BL` a partir dos call sites do início da imagem.
2. **Eles apontam para a PLT** — 100% deles, nos dois binários. Não é um desvio para qualquer lugar: é o caminho da PLT sendo redirecionado.
3. **A distância é o gatilho**: a PLT começa a 33.624.752 bytes do início do `.text` no build bom e a 33.658.080 no que quebrou — ou seja, **68,7 KB e 101,2 KB além** dos 33.554.432 bytes de alcance do `BL`. Margem ínfima, o que explica por que 42 KB de código novo mudaram tanto: de 132 para 391 call sites afetados.

### Experimento controlado (2026-08-24)

Os dois APKs assinados da investigação sobreviveram (167 MB cada, com o `libaot-AsaasMobile.dll.so` de 32 bits dentro). Foram instalados em sequência no mesmo aparelho do incidente — **Samsung SM-J510MN, `armeabi-v7a`, Android 7.1.1** — os dois com AOT:

| Build | `.text` | PLT dentro do `.text` | Veneers | Resultado no aparelho |
|---|---:|---:|---:|---|
| [`92cb32ab`](https://github.com/asaasdev/asaas-mobile/commit/92cb32ab) | 41,74 MiB | +32,07 MiB (68,7 KB além do alcance) | 132 | **sobe** — processo vivo, nenhum assert |
| [`4bdcb74f`](https://github.com/asaasdev/asaas-mobile/commit/4bdcb74f) | 41,79 MiB | +32,10 MiB (101,2 KB além) | 391 | **aborta** no boot |

O log do segundo, capturado na hora:

```
E         : * Assertion at /__w/1/s/src/mono/mono/mini/aot-runtime.c:5339, condition `plt_entry' not met
F monodroid: * Assertion at /__w/1/s/src/mono/mono/mini/aot-runtime.c:5339, condition `plt_entry' not met
--------- beginning of crash
F libc    : Fatal signal 6 (SIGABRT), code -6 in tid 30165 (asaas.asaas)
F DEBUG   : Abort message: '* Assertion at ...aot-runtime.c:5339, condition `plt_entry' not met
```

É o A/B que faltava desde 2026-08-13, quando só se havia instalado um build **sem** AOT. Três conclusões que ele fixa:

1. O commit apontado pelo `git bisect` **não** era falso positivo: com AOT, um build sobe e o outro não, e a diferença entre eles é de 42 KB de código.
2. **Ter veneers não basta para quebrar.** O build que sobe tem 132 chamadas de PLT passando por veneer e funciona — porque nenhuma delas roda no boot. É a confirmação direta da dependência de layout.
3. O que muda o resultado é o *conjunto* de chamadas afetadas: 132 → 391 foi suficiente para pegar uma do caminho de inicialização.

### O lado do runtime: o que o código faz

Do `src/mono/mono/mini/aot-runtime.c` do .NET 9 (`release/9.0`) — a linha 5339 citada no crash é literalmente o `g_assert`:

```c
// mono_aot_plt_resolve(), linha 5338-5339
plt_entry = mono_aot_get_plt_entry (regs, code);
g_assert (plt_entry);
```

E `mono_aot_get_plt_entry()` decodifica a instrução do call site e valida a faixa:

```c
target = mono_arch_get_call_target (code);      // le o BL do call site e extrai o destino
...
if ((target >= (guint8*)(amodule->plt)) && (target < (guint8*)(amodule->plt_end)))
    return target;
else
    return NULL;                                 // destino fora da PLT -> NULL -> assert
```

Não sobra ambiguidade: quando o `BL` aponta para o veneer, `target` cai fora de `[plt, plt_end]`, a função devolve `NULL` e o assert derruba o processo.

**Detalhe que vale registrar:** o mesmo arquivo tem, sob `#ifdef TARGET_APPLE_MOBILE`, um laço que **segue a cadeia** — se o destino não está na PLT, ele decodifica o branch em `target + 4` e tenta de novo, com o comentário explicando que é para pular o desvio intermediário. Ou seja, o upstream já trata exatamente este caso, mas só compila esse caminho para Apple mobile; o Android cai no `#else`, que devolve `NULL` direto.

Com isso, a cadeia até o assert fica completa: o call site do boot faz `BL` para o veneer em vez da entrada da PLT; o runtime, ao resolver a PLT, decodifica a instrução do call site esperando recuperar o endereço de uma entrada da PLT, obtém o endereço do veneer, que está fora da faixa da PLT, e `plt_entry` volta `NULL` → assert → `SIGABRT`.

### Por que o `.text` do assembly principal era tão grande

Este é o ponto que explica o crescimento, e ele **não** vale para os outros assemblies da solução: o SDK do Android aplica o filtro de profile do AOT a todos os assemblies **exceto ao assembly principal**. Na inspeção do `Microsoft.Android.Sdk.Darwin/35.0.105`, o próprio SDK registra:

```
Not using profile(s) for main assembly
```

Resultado: todos os outros projetos compilam antecipadamente só os métodos do profile de inicialização e deixam o resto para o JIT; o assembly principal compila **tudo**.

| Assembly | IL (C#) | Nativo AOT | Expansão | Recebe o filtro de profile? |
|---|---:|---:|---:|---|
| `AsaasMobile.dll` (principal) | 14,3 MB | 43,9 MB | **3,07×** | **não** |
| `Asaas.Framework.dll` | 1,7 MB | 0,5 MB | 0,29× | sim |

**Experimento de controle (o que fecha a conclusão):** um build com `AndroidEnableProfiledAot=false` produziu o `AsaasMobile.dll.so` **byte a byte idêntico** (44.906.136 bytes) ao build com o profile ligado. O filtro nunca teve efeito nenhum sobre o assembly principal — ligado ou desligado, ele sempre foi compilado por inteiro.

### Por que o PR "culpado" não tem nada a ver

O limiar é atingido por **acumulação**. Qualquer commit que apenas adiciona código pode ser o que empurra o `.text` para dentro da faixa de falha — no caso de 2026-08-13, o commit apontado pelo bisect adicionou 42 KB a um `.text` que já tinha 41,7 MiB. O diff é inocente e não existe bug lógico para achar.

O commit apontado pelo bisect de fato **cruzou** o ponto de falha (ver a tabela por commit em [Evidência medida](#evidência-medida)) — o que não significa que revertê-lo seja correção: isso devolveria o app ao mesmo ponto de fragilidade, com 42 KB de folga e sujeito ao layout do próximo build.

## Evidência medida

Device: Samsung SM-J510MN, `ro.product.cpu.abi = armeabi-v7a`. Packs: `Microsoft.Android.Sdk.Darwin 35.0.105` / `Microsoft.Android.Runtime.35.* 35.0.105`.

Tamanhos no APK (`asaas.asaas-Signed.apk`) do build que crashou:

```
45.231.224  lib/armeabi-v7a/libaot-AsaasMobile.dll.so   <-- estoura
41.128.624  lib/arm64-v8a/libaot-AsaasMobile.dll.so     <-- ok (±128 MiB)
 2.295.992  lib/armeabi-v7a/libaot-Microsoft.Maui.Controls.dll.so
```

Seções do ELF de `armeabi-v7a/libaot-AsaasMobile.dll.so`:

```
.text          44.134.256 bytes  (~42,1 MiB)   addr=0x86d60
.rel.dyn          547.768
.data.rel.ro      547.568
.bss              491.392
```

**42,1 MiB de `.text` contra um teto físico de 32 MiB.** O `AsaasMobile.dll` era o único assembly que estourava — o segundo maior módulo AOT era o `System.Private.CoreLib`, com cerca de 3 MB. O limite é **por `.so`**, não do app inteiro.

`.text` de `armeabi-v7a` em dois builds limpos, antes e depois do range apontado pelo bisect:

| Commit | `.text` (`armeabi-v7a`) | Boot em 32 bits |
|---|---:|---|
| `92cb32ab` (antes) | 43.775.800 (41,7 MiB) | **sobe** — verificado no aparelho em 2026-08-24 |
| `4bdcb74f` (depois) | 43.818.376 (41,8 MiB) | **aborta** — verificado no aparelho em 2026-08-24 |
| build que gerou o log de crash | 44.134.256 (42,1 MiB) | **aborta** |

Duas leituras importam aqui: a diferença entre subir e abortar foi de **42.576 bytes**, e o build que produziu o log de crash não é nenhum dos dois medidos — tem ~316 KB de código a mais, provavelmente `master`. **Ao reproduzir, registre o commit exato de cada build.**

> **Nota.** Até 2026-08-24 o "sobe" do `92cb32ab` vinha só do resultado do `git bisect` — o que se havia instalado no aparelho era um build **sem** AOT. Os dois APKs com AOT foram finalmente instalados lado a lado no mesmo SM-J510MN, confirmando a linha da tabela; ver [Experimento controlado](#experimento-controlado-2026-08-24).

Experimento de controle do profile de AOT, no mesmo commit:

| Build | `AsaasMobile.dll.so` |
|---|---:|
| `AndroidEnableProfiledAot=true` (default) | 44.906.136 |
| `AndroidEnableProfiledAot=false` | 44.906.136 — **byte a byte idêntico** |

## Como diagnosticar em 2 minutos

1. **Confirmar a ABI do device** — se for arm64, este documento não se aplica:

```bash
adb shell getprop ro.product.cpu.abi
adb shell getprop ro.product.model
```

2. **Rodar o guard** (sem rebuildar nada; use o `obj` do build que crashou). Ele mede a condição exata — posição da PLT e veneers apontando para ela — e o tamanho do `.text` como rede de segurança. O glob de TFM evita hardcode, já que o alvo mudou de `net9.0-android` para `net10.0-android` na migração em andamento:

```bash
python3 build-files/android/check_aot_text_size.py \
  App/AsaasMobile/obj/Release/net*.0-android/android-arm/aot/
```

```
Assembly                                     Arch    .text (MiB)  PLT (+MiB)    margem  veneers  Status
AsaasMobile.dll.so                           arm32         41.79       32.10     -0.10      391  FALHA (391 veneers -> PLT)
```

**Como ler:** `PLT (+MiB)` é a distância da PLT ao início do `.text`; `margem` é o que sobra até os 32 MiB de alcance do `BL`. Margem negativa com veneers apontando para a PLT é o diagnóstico fechado — a imagem está no regime de falha, mesmo que ela suba. Margem positiva e zero veneers descartam este mecanismo.

Sai com código 1 se alguma imagem 32-bit estiver nesse estado, e 2 se não encontrar nenhum `*.dll.so` (caminho errado não passa em silêncio). Imagens de 64 bits são listadas mas nunca reprovam.

Se você estiver num checkout antigo, sem o script, o mesmo cálculo para um único `.so`:

```bash
python3 - App/AsaasMobile/obj/Release/net9.0-android/android-arm/aot/AsaasMobile.dll.so <<'EOF'
import struct, sys

data = open(sys.argv[1], 'rb').read()
assert data[:4] == b'\x7fELF', "nao e um ELF"
is64 = data[4] == 2

if is64:
    e_shoff, = struct.unpack_from('<Q', data, 0x28)
    e_shentsize, e_shnum, e_shstrndx = struct.unpack_from('<HHH', data, 0x3A)
    layout = '<IIQQQQ'
else:
    e_shoff, = struct.unpack_from('<I', data, 0x20)
    e_shentsize, e_shnum, e_shstrndx = struct.unpack_from('<HHH', data, 0x2E)
    layout = '<IIIIII'

def section(index):
    name, typ, flags, addr, offset, size = struct.unpack_from(layout, data, e_shoff + index * e_shentsize)
    return name, offset, size

strtab_offset = section(e_shstrndx)[1]

for index in range(e_shnum):
    name, _, size = section(index)
    end = data.index(b'\0', strtab_offset + name)
    if data[strtab_offset + name:end] == b'.text':
        print(f".text = {size:,} bytes ({size / 1024 / 1024:.1f} MiB) — teto fisico ARM32: 32 MiB")
EOF
```

O script funciona para ELF32 (`armeabi-v7a`) e ELF64 (`arm64-v8a`).

Ou, direto no APK:

```bash
unzip -l App/AsaasMobile/bin/Release/net*.0-android/asaas.asaas-Signed.apk \
  | grep -E "libaot-.*\.so" | sort -rn | head
```

**Como ler o número.** Acima de 32 MiB em `armeabi-v7a`, esta é a hipótese principal — mas o tamanho sozinho **não fecha nem descarta** o diagnóstico: 41,7 MiB subiu e 41,8 MiB abortou no mesmo device. A confirmação vem dos dois experimentos da seção seguinte. Abaixo de 32 MiB, procure outra causa.

3. **Log do runtime AOT** (opcional, para ver qual módulo estava sendo resolvido):

```bash
adb shell setprop debug.mono.log assembly,aot
adb logcat -c && adb logcat | grep -iE "monodroid|mono-rt|aot"
```

4. **Ver se o filtro de profile está sendo aplicado** ao assembly que estourou — se aparecer a linha abaixo no log do build, aquele assembly está compilando 100% dos métodos:

```bash
# net10.0-android apos a migracao em andamento; confira com: ls -d App/AsaasMobile/obj/Release/net*.0-android
dotnet build App/AsaasMobile/AsaasMobile.csproj -c Release -f net9.0-android -v n \
  | grep -i "profile"
# "Not using profile(s) for main assembly" = o assembly principal ignora o profile
```

## Como confirmar

Dois experimentos independentes, ambos baratos:

- **Rebuildar com AOT desligado** → crash desaparece no mesmo device 32-bit:

```bash
# net10.0-android apos a migracao em andamento; confira com: ls -d App/AsaasMobile/obj/Release/net*.0-android
dotnet build App/AsaasMobile/AsaasMobile.csproj -c Release -f net9.0-android \
  -p:SuppressWarnings=true -p:RunAOTCompilation=false
```

- **Instalar o mesmo APK (com AOT) num device arm64** → deve subir normalmente. Confirma que é específico da ABI, não do código.

Comandos úteis para verificar o que está de fato instalado no device:

```bash
# ABI usada pelo processo e horário do último install
adb shell dumpsys package asaas.asaas | grep -iE "primaryCpuAbi|lastUpdateTime|versionName"

# o build instalado tem imagens AOT? (com AOT: aparecem varios libaot-*.so)
adb shell ls /data/app/asaas.asaas-1/lib/arm/ | grep -i aot

# crashes registrados, com timestamp
adb logcat -b crash -d | grep -iE "plt_entry|Assertion at"
```

### Observado em 2026-08-13 (SM-J510MN, `armeabi-v7a`)

Dois builds Release do mesmo dia, no mesmo device — **o commit exato de cada um não foi registrado**, então trate como duas observações, não como experimento controlado:

| Build | `build.props` | Resultado no device 32-bit |
|---|---|---|
| com AOT | `aotassemblies=true`, `androidaotmode=normal` | assert `plt_entry` + `SIGABRT` no boot (11:41 e 11:48) |
| sem AOT | `aotassemblies=false`, `androidaotmode=none` | sobe normalmente (install 12:12:33, boot 12:12:59, nenhum crash) |

O install sem AOT não tem nenhum `libaot-*.so` em `/data/app/asaas.asaas-1/lib/arm/` — apenas `libmonodroid.so` e `libmonosgen-2.0.so`. Combinado com o `.text` medido acima, aponta a causa para a compilação AOT, não para o código de feature.

## O que NÃO fazer

- **Não bisectar antes de medir o `.text`.** Cada build Release com AOT gera dezenas de MB de imagem nativa e leva vários minutos, e se o `.text` estiver na faixa de falha o commit que o bisect aponta é apenas o que cruzou o limiar — não a causa.
- **Não reverter o commit apontado pelo bisect** achando que é correção. Ele devolve o app ao ponto de fragilidade anterior, com dezenas de KB de folga.
- **Não procurar bug lógico no diff.** O crash acontece antes do código novo executar.
- **Não tratar "abaixo de 42 MiB" como seguro, nem "subiu" como prova de saúde.** O build de 41,74 MiB sobe no aparelho e mesmo assim tem 132 chamadas de PLT passando por veneer. A grandeza que importa é a distância da PLT ao início do `.text`, não o tamanho total.
- **Não tentar resolver o assembly principal com profile de AOT customizado** (`AndroidAotProfile`, `AndroidEnableProfiledAot`). O SDK ignora profile no main assembly — comprovado pelo controle byte a byte acima.
- **Não atribuir a artefato stale.** Vale checar (`ls -lT obj/Release/net*.0-android/android-arm/aot`), mas se todos os `.so` têm o mesmo timestamp, o build é consistente e staleness está descartado.

## Correção aplicada

**O que foi feito (2026-08-18):** todo o código do app foi movido para a class library `AsaasMobile.Core`, e o `AsaasMobile.csproj` ficou reduzido a uma casca — ponto de entrada, `Platforms/`, `MauiProgram` e assets. Ao deixar de ser o assembly principal, o código do app passa a receber o filtro de profile do AOT como qualquer outra biblioteca: o AOT continua rodando, mas compila apenas os métodos do profile de inicialização e deixa o resto para o JIT.

O passo a passo, as armadilhas (NU1107 do `Xamarin.AndroidX.Fragment`, `Assembly.GetExecutingAssembly()`, geradores de `Resources/Strings`) e o que fica na casca estão em `docs/asaas-mobile-core-migration.md`.

Medição em dois builds Release completos do mesmo commit base (`72249ca9f5`), em `armeabi-v7a`:

| Biblioteca nativa | Arquivo | `.text` |
|---|---:|---:|
| **antes** — `AsaasMobile.dll.so` | 45.275.608 | 44.177.896 (**42,1 MiB**) |
| **depois** — `AsaasMobile.dll.so` (casca) | 85.968 | 80.864 (0,08 MiB) |
| **depois** — `AsaasMobile.Core.dll.so` | 2.521.912 | 1.424.744 (**1,4 MiB**) |

Depois da mudança, a maior imagem AOT de todo o app passou a ser a do próprio .NET (`System.Private.CoreLib`), com 2,6 MiB contra um teto de 32 MiB. O total caiu de 59,1 para 18,4 MiB por arquitetura. A falha deixa de ser possível **por construção**, não por margem.

**Condição que a correção exige:** o filtro de profile precisa continuar ligado (`AndroidEnableProfiledAot=true`, sem `AndroidUseDefaultAotProfile=false`). Se alguém desligar, o `AsaasMobile.Core` volta a compilar tudo e o crash reaparece. Atenção: **nada no repositório fixa esse valor** — não há `AndroidEnableProfiledAot` em nenhum `.csproj`, `.props` ou `.targets`; o app depende do default do SDK para Release. Um argumento de linha de comando ou uma variável do pipeline bastam para derrubar a correção sem nenhum diff no código.

**Trade-off em aberto:** o código movido passa a ser compilado sob demanda (JIT). A redução de 96,8% mede o custo do passo de compilação, não a experiência do app — cold start e tempo de primeira tela ainda precisam ser medidos.

### Outras opções, e por que não

| Opção | Veredito |
|---|---|
| `<RuntimeIdentifiers>android-arm64</RuntimeIdentifiers>` (dropar 32-bit) | resolve de vez, correção de uma linha, mas perde devices 32-bit-only — o Play deixa de servir o app para eles. Hoje o projeto declara `android-arm;android-arm64`. Não reduziria o download de ninguém: com `androidpackageformat=aab` o Play já entrega só a slice da ABI do device |
| `RunAOTCompilation=false` | resolve em todas as ABIs, mas com regressão de cold start para 100% da base. Serve como experimento de confirmação, não como correção |
| Profile de AOT customizado (`AndroidAotProfile`) no assembly principal | **não funciona** — o SDK ignora profile no main assembly (controle byte a byte acima) |
| Quebrar `AsaasMobile.dll` em assemblies menores por feature | funcionaria, mas o mecanismo não é "assemblies menores" e sim "deixar de ser o assembly principal" — é a mesma ideia da correção aplicada, com mais trabalho |
| Remover código/feature para voltar abaixo do limiar | **paliativo**, usado durante o incidente para desbloquear a loja. Devolve margem sem mudar nada da causa: o próximo volume de código consome a folga de novo |

## Prevenção

O `.text` do maior `libaot-*.so` em `armeabi-v7a` é uma métrica de build que só sobe, e o custo de descobrir o estouro em produção é um crash 100% reproduzível no boot, sem stack trace gerenciado — invisível para qualquer teste feito só em devices de 64 bits.

Estado atual da prevenção, com o que falta em cada frente:

| Item | Estado |
|---|---|
| Guard (`build-files/android/AotTextSizeGuard.targets` + `check_aot_text_size.py`) | **ligado ao build e validado** — target `CheckAndroidAotTextSize`, `AfterTargets="_AndroidAot"`, exercitado nos dois caminhos em 2026-08-24 (ver [Onde ligar o guard](#onde-ligar-o-guard)). Não havia como usar o GitHub Actions para isso: o build Android de Release não acontece lá — `.github/workflows` tem atlas-report, auto-bump-versions, merge-master-into-release, pull_request_toolkit, release-drafter, static-analysis e a action composta `version-autobump`; o único que compila é o `static-analysis.yml`, com `dotnet build` em Debug para o Reviewdog, e sem `-c Release` não há imagem AOT para medir. As duas formas de ligar o guard estão descritas abaixo |
| Critério do guard | **derivado, não mais arbitrário.** Reprova quando a PLT passa dos 32 MiB do início do `.text` ou quando há veneer apontando para ela — a condição de falha em si, medida da imagem. Os limites de tamanho (28 alerta / 30 falha) ficaram como rede de segurança, e há um alerta novo por **margem de PLT** abaixo de 8 MiB. Validado contra os dois binários do incidente: reprova os dois, inclusive o que sobe |
| Teste de boot em 32 bits | **não existe.** A validação de release é feita só em devices de 64 bits, e por isso a fatia do pacote que quebrava nunca foi executada internamente. Um único boot em qualquer device ou emulador `armeabi-v7a` teria barrado a publicação |
| Alerta de crash no boot | **não existe.** No incidente, os chamados de suporte foram a única fonte de detecção: 5 dias entre a publicação e o primeiro relato. Vale um alerta segmentado por arquitetura e versão a partir do Android vitals |

### Onde ligar o guard

O pipeline de release roda no Azure DevOps no **modelo clássico**: a definição fica na própria interface do Azure, **não** há `azure-pipelines.yml` neste repositório. Isso dá duas opções, com trade-offs diferentes.

**Opção 1 — step no pipeline clássico (Azure DevOps).** Adicionar uma task de script na definição do pipeline, logo após o build/publish Release Android e antes do empacotamento em AAB:

```bash
python3 build-files/android/check_aot_text_size.py \
  "$(Build.SourcesDirectory)/App/AsaasMobile/obj/Release/net9.0-android/android-arm/aot/"
```

O que considerar nesse caminho: a configuração vive fora do repositório, então não passa por revisão de PR, não aparece em diff e ninguém descobre pelo código se o step for removido depois — a existência do guard passa a depender de quem tem permissão de editar o pipeline. Por isso importa o comportamento de saída do script: `exit 2` quando não encontra nenhum `.dll.so` no caminho, para que um caminho errado na configuração falhe em vez de passar em silêncio como "nenhum risco encontrado".

**Opção 2 — target MSBuild versionado no repositório — _foi a aplicada_.** A validação vive em `build-files/android/AotTextSizeGuard.targets`, ao lado do script que ela executa, e o `AsaasMobile.csproj` só a importa:

```xml
<!-- AsaasMobile.csproj -->
<Import Project="$(MSBuildThisFileDirectory)../../build-files/android/AotTextSizeGuard.targets" />
```

```xml
<!-- AotTextSizeGuard.targets -->
<Target Name="CheckAndroidAotTextSize"
        AfterTargets="_AndroidAot"
        Condition="'$(AotAssemblies)' == 'true' AND '$(RuntimeIdentifier)' != '' AND '$(SkipAndroidAotTextSizeCheck)' != 'true'">
  <Exec Command="python3 &quot;$(AndroidAotTextScript)&quot; &quot;$(_AndroidAotBinDirectory)&quot;..." />
</Target>
```

O `.targets` acha o script por `$(MSBuildThisFileDirectory)`, então guard e script se movem juntos, e o csproj fica com uma linha. Para rodar a verificação **sem buildar**, sobre um `obj/` já existente:

```bash
dotnet msbuild App/AsaasMobile/AsaasMobile.csproj -t:CheckAndroidAotTextSize \
  -p:Configuration=Release -p:TargetFramework=net9.0-android \
  -p:RuntimeIdentifier=android-arm -p:AotAssemblies=true
```

Por que `_AndroidAot` e não `_RunAotForAllRIDs`: o segundo só executa quando `_AndroidUseMarshalMethods == True` (verificado em `Xamarin.Android.Common.targets` dos packs 34.0.154, 35.0.105 e 36.1.69), então serviria de gancho silenciosamente inerte. O `_AndroidAot` roda no build interno de cada RID, com a condição `'$(AotAssemblies)' == 'true' and '$(RuntimeIdentifier)' != ''` — exatamente quando existe imagem AOT para medir. O diretório vem do próprio SDK: `_AndroidAotBinDirectory` = `$(IntermediateOutputPath)aot`, ou seja `obj/Release/net9.0-android/android-arm/aot` no build de 32 bits.

Ajustes disponíveis sem editar o script:

```bash
# apertar as redes de seguranca por tamanho (o criterio exato de falha nao depende delas)
dotnet build ... -p:AndroidAotTextWarnMiB=4 -p:AndroidAotTextFailMiB=8

# alertar mais cedo pela margem de PLT (default: alerta com menos de 8 MiB de folga)
dotnet build ... -p:AndroidAotTextMarginWarnMiB=12

# encurtar o log: listar so as N maiores imagens (default 0 = todas)
dotnet build ... -p:AndroidAotTextTop=10

# nao emitir o warning quando passa (a tabela volta a sair so em -tl:off / -v n)
dotnet build ... -p:AndroidAotTextQuiet=true

# pular a verificação
dotnet build ... -p:SkipAndroidAotTextSizeCheck=true
```

Vantagens sobre a Opção 1: roda onde quer que o build de Release aconteça, incluindo o agente do Azure, sem depender da configuração da interface, e a proteção passa a ser revisável por PR junto com o código. É o mesmo padrão que o projeto já usa para as ferramentas de strip do iOS (`build-files/ios/{bitcode_strip,ld}`, chamadas por um target no `AsaasMobile.csproj`).

O que isso passa a exigir do ambiente de build: `python3` no PATH do agente. Se faltar, o build de Release Android quebra no `Exec` — de propósito, para não virar verificação silenciosamente ausente; o escape é o `SkipAndroidAotTextSizeCheck`.

**Onde ver o resultado.** O terminal logger do .NET (ligado por padrão quando a saída é um terminal) mostra **apenas warnings, errors e a linha de resumo do projeto** — `Message` e stdout de `Exec` não aparecem, por mais alta que seja a importância. Verificado em projeto de teste: sob o terminal logger, uma `Message Importance="high"` e a saída de um `Exec` com `StandardOutputImportance="High"` somem; as mesmas linhas aparecem com `-tl:off` ou quando a saída é redirecionada. Como warning e error são os únicos canais confiáveis, é por eles que o guard fala:

| Situação | Canal |
|---|---|
| Passou (default) | `<Warning AOT32>` com a tabela junto — aparece num `dotnet build` sem flag nenhuma |
| Passou, com `-p:AndroidAotTextQuiet=true` | Nenhum warning; a tabela sai no stdout, visível só com `-tl:off` / `-v n` |
| Falha de limite (`exit 1`) | `<Error>` com a tabela junto |
| Erro de ferramenta (outros exits) | `<Error>` dizendo se faltou `*.dll.so` no caminho ou `python3` no PATH |
| Sempre, em qualquer caso | Tabela completa em `obj/Release/net*.0-android/<rid>/aot-text-size.log`, uma por RID |

Saída real de um build que passa, sob o terminal logger:

```
AsaasMobile net9.0-android android-arm succeeded with 1 warning(s) (0,2s)
  AsaasMobile.csproj(421,9): warning AOT32:
    [guard AOT] android-arm: tamanho das imagens AOT dentro do limite. Tabela completa em obj/Release/net9.0-android/android-arm/aot-text-size.log
      Assembly                                     Arch    .text (MiB)  PLT (+MiB)    margem  veneers  Status
      System.Private.CoreLib.dll.so                arm32          2.60        1.71    +30.29        0  ok
      Microsoft.Maui.Controls.dll.so               arm32          1.98        1.25    +30.75        0  ok
      AsaasMobile.Core.dll.so                      arm32          1.39        0.35    +31.65        0  ok
      [... as demais imagens ...]
      Total 32-bit: 223 imagens, .text somado = 14.0 MiB (o limite e por arquivo, nao pela soma)
      Menor margem de PLT: System.Private.CoreLib.dll.so, PLT a 1.71 MiB do inicio do .text (+30.29 MiB ate o alcance do BL)
      OK: nenhuma imagem AOT 32-bit no regime de falha.
```

Quando reprova, o mesmo conteúdo sai como `error` em vez de `warning`, com o link do runbook e a flag de ajuste do limite.

O default lista **todas** as imagens (~230 linhas por RID); no pipeline convém `-p:AndroidAotTextTop=15`, que encurta a tabela sem perder totais nem veredito — o arquivo de relatório continua completo.

**Validação (2026-08-24).** Rodou em build de Release completo (`net9.0-android`, os dois RIDs) nos dois caminhos: com os limites default passou, com `-p:AndroidAotTextFailMiB=1` reprovou o build. Os dois canais foram conferidos sob o terminal logger invocando o target isolado (`dotnet msbuild -t:CheckAndroidAotTextSize`). Estado medido nesse build: maior imagem 32-bit `System.Private.CoreLib.dll.so` com **2,6 MiB**, seguida de `Microsoft.Maui.Controls.dll.so` (2,0 MiB) e `AsaasMobile.Core.dll.so` (**1,4 MiB**), contra um teto físico de 32 MiB. O `build.props` do mesmo build confirma `aotassemblies=true`, `androidaotmode=normal` e `androidenableprofiledaot=true` — o filtro de profile está ativo e o código do app, agora fora do assembly principal, o recebe.

Com o guard rodando em todo build de Release, a reincidência silenciosa fica coberta; o que continua descoberto é o boot em si — nenhum teste executa o app numa arquitetura de 32 bits, que foi exatamente a lacuna que deixou o incidente chegar à loja.

## Glossário

| Termo | O que significa |
|---|---|
| **AOT** (*Ahead-of-Time*) | Compilar o código C# para código de máquina nativo já no build. O app abre mais rápido e o pacote fica maior. |
| **JIT** (*Just-in-Time*) | O oposto: compilar em tempo de execução, na primeira vez que cada método roda. Não engorda o pacote, mas a primeira execução de cada método é mais lenta. |
| **assembly** | Um arquivo `.dll`, a unidade de compilação do .NET. O **assembly principal** é o do projeto que produz o app — e o SDK do Android trata ele de forma diferente de todos os outros, que é a origem deste incidente. |
| **`.text`** | Dentro de uma biblioteca nativa, a seção que guarda o código executável. |
| **ABI / `armeabi-v7a`** | A variante de código de máquina que o aparelho entende. `armeabi-v7a` = ARM de 32 bits; `arm64-v8a` = ARM de 64 bits. |
| **PLT** | Tabela de "trampolins" que o código usa para chamar funções cujo endereço só se conhece durante a execução. No AOT do Mono ela fica dentro do próprio `.text`, **depois do código dos métodos** — por isso quanto mais código, mais longe ela fica de quem precisa alcançá-la. |
| **veneer** (*range-extension thunk*) | O desvio intermediário que o linker insere quando o destino de uma chamada está longe demais. Resolve o problema do linker e quebra o runtime, que esperava a chamada apontando direto para a PLT. |
| **profile de AOT** | Lista dos métodos que valem a pena compilar antecipadamente (os que rodam na inicialização). Com ela aplicada, o AOT compila só o que está na lista e deixa o resto para o JIT. |
| **class library** | Projeto que gera apenas uma `.dll`, sem ser executável. É para onde o código do app foi movido na correção definitiva. |

## Rastreabilidade

- Chamado do incidente: <https://asaasdev.atlassian.net/browse/DDST-1810>
- Correção paliativa (remoção de feature não usada): <https://github.com/asaasdev/asaas-mobile/pull/2208> — revertida por <https://github.com/asaasdev/asaas-mobile/pull/2217>
- Guard de `.text` + primeira versão deste runbook: <https://github.com/asaasdev/asaas-mobile/pull/2206>
- Limpeza de código e imagens não utilizados: <https://github.com/asaasdev/asaas-mobile/pull/2226>
- Correção definitiva: <https://github.com/asaasdev/asaas-mobile/pull/2228> (criação do `AsaasMobile.Core`) e <https://github.com/asaasdev/asaas-mobile/pull/2229> (movimentação dos arquivos)
- Branch original da correção definitiva: `refactor/app-core-library`, base [`72249ca9f5`](https://github.com/asaasdev/asaas-mobile/commit/72249ca9f5)
  - [`348b6db77c`](https://github.com/asaasdev/asaas-mobile/commit/348b6db77c) — `build: add AsaasMobile.Core class library project`
  - [`af2006804d`](https://github.com/asaasdev/asaas-mobile/commit/af2006804d) — `refactor: move app code out of the main assembly into AsaasMobile.Core`
- Commits usados nas medições comparativas: [`92cb32ab`](https://github.com/asaasdev/asaas-mobile/commit/92cb32ab) (subia) e [`4bdcb74f`](https://github.com/asaasdev/asaas-mobile/commit/4bdcb74f) (abortava); base da correção definitiva: [`72249ca9f5`](https://github.com/asaasdev/asaas-mobile/commit/72249ca9f5)
- Versão de produção afetada: tag `v558.0`
- Migração para `AsaasMobile.Core`: `docs/asaas-mobile-core-migration.md`
- Guard: `build-files/android/AotTextSizeGuard.targets` (integração com o build) e `build-files/android/check_aot_text_size.py` (medição e diagnóstico)
