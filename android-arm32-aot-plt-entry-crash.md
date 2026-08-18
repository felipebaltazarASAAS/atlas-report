# Crash no boot em Android 32-bit — `aot-runtime.c ... condition 'plt_entry' not met`

> Guia de diagnóstico para um crash de inicialização que **não** tem causa no código do PR. Se você chegou aqui vindo de um `git bisect`, pare o bisect e leia a seção [Como diagnosticar em 2 minutos](#como-diagnosticar-em-2-minutos).

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
- Reproduz em `armeabi-v7a`; em `arm64-v8a` não deve reproduzir (a checar caso a caso — o alcance do branch em A64 é 4× maior).
- Aparece "a partir de um commit" cujo diff não tem nada relacionado a startup, DI, XAML de App ou runtime.

## Causa raiz

O `.text` da imagem AOT de um assembly passou do alcance do branch direto do ARM 32-bit.

O build Release do Android usa AOT (veja `obj/Release/net9.0-android/build.props`: `aotassemblies=true`, `androidaotmode=normal`, `androidenableprofiledaot=true`, `androidlinkmode=sdkonly`). Cada assembly gera uma biblioteca nativa `lib/<abi>/libaot-<Assembly>.dll.so`, contendo o código nativo dos métodos **e** a PLT do módulo, na mesma seção `.text`.

Limites de alcance de branch:

| Arquitetura | Instrução | Alcance |
|---|---|---|
| ARM 32-bit (A32) | `BL` | ±32 MiB |
| ARM 32-bit (Thumb-2) | `BL` | ±16 MiB |
| ARM 64-bit (A64) | `BL` | ±128 MiB |

Quando o `.text` cresce além desse alcance, o linker (lld) insere *range-extension thunks* (veneers) para as chamadas que não alcançam mais o destino. A partir daí, o call site aponta para o veneer, não para a entrada da PLT.

O runtime, no caminho de resolução de PLT (`aot-runtime.c`), **decodifica a instrução do call site** para recuperar o endereço da entrada da PLT que o chamou. Com o veneer no caminho, o endereço calculado não cai dentro da faixa da PLT do módulo, o resultado é `NULL`, e o assert `plt_entry` falha → `SIGABRT`.

Como o crash acontece na primeira chamada AOT "longe" o suficiente, ele cai no boot.

> **Nível de certeza.** Medidos: o tamanho do `.text` acima do alcance do ARM32 e a dependência do AOT (ver [Evidência](#evidência-do-caso-de-2026-08-13) e [Observado](#observado-em-2026-08-13-sm-j510mn-armeabi-v7a)). Inferidos, a partir do assert: o veneer inserido pelo lld quebrando o reconhecimento do call site, e a posição da PLT dentro do `.text` — o `.so` é stripped, não foi desmontado o binário para ver o thunk nem os símbolos. Nada disso muda as opções de correção.

### Por que o PR "culpado" não tem nada a ver

O limiar é atingido por **acumulação**. Qualquer commit que apenas adiciona código pode ser o que empurra o `.text` de 31,9 MiB para 32,1 MiB — o diff que o bisect aponta é inocente e não existe bug lógico para achar.

> Esta parte é **inferida**: o `.text` de 42,1 MiB foi medido no build que crashou, mas ninguém mediu o `.text` do commit "bom" (`92cb32ab`, no caso de 2026-08-13). A medição que fecharia a história é um build AOT desse commit: se der abaixo de 32 MiB, o cruzamento do limiar está provado; se já vier acima, o mecanismo é outro.

## Evidência do caso de 2026-08-13

Device: Samsung SM-J510MN, `ro.product.cpu.abi = armeabi-v7a`. Packs: `Microsoft.Android.Sdk.Darwin 35.0.105` / `Microsoft.Android.Runtime.35.* 35.0.105`.

Tamanhos no APK (`asaas.asaas-Signed.apk`):

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

**42,1 MiB de `.text` contra um alcance de 32 MiB.** O `AsaasMobile.dll` é o único assembly que estoura: o segundo maior módulo AOT é o `System.Private.CoreLib` com 3 MB. O limite é **por `.so`**, não do app inteiro.

## Como diagnosticar em 2 minutos

1. **Confirmar a ABI do device** — se for arm64, este documento não se aplica:

```bash
adb shell getprop ro.product.cpu.abi
adb shell getprop ro.product.model
```

2. **Medir o `.text` da imagem AOT da ABI que crasha** (sem rebuildar nada; use o `obj` do build que crashou):

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
        print(f".text = {size:,} bytes ({size / 1024 / 1024:.1f} MiB) — limite ARM32: 32 MiB")
EOF
```

O script funciona para ELF32 (`armeabi-v7a`) e ELF64 (`arm64-v8a`).

Ou, direto no APK:

```bash
unzip -l App/AsaasMobile/bin/Release/net9.0-android/asaas.asaas-Signed.apk \
  | grep -E "libaot-.*\.so" | sort -rn | head
```

Se `libaot-<Assembly>.dll.so` de `armeabi-v7a` estiver na casa dos 32 MB+, o diagnóstico está fechado.

3. **Log do runtime AOT** (opcional, para ver qual módulo estava sendo resolvido):

```bash
adb shell setprop debug.mono.log assembly,aot
adb logcat -c && adb logcat | grep -iE "monodroid|mono-rt|aot"
```

## Como confirmar

Dois experimentos independentes, ambos baratos:

- **Rebuildar com AOT desligado** → crash desaparece no mesmo device 32-bit:

```bash
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

- **Não bisectar antes de medir o `.text`.** Cada build Release com AOT gera ~45 MB de imagem nativa e leva vários minutos, e se o `.text` estiver acima do limite o commit que o bisect aponta é apenas o que cruzou o limiar — não a causa.
- **Não procurar bug lógico no diff.** O crash acontece antes do código novo executar.
- **Não atribuir a artefato stale.** Vale checar (`ls -lT obj/Release/net9.0-android/android-arm/aot`), mas se todos os `.so` têm o mesmo timestamp, o build é consistente e staleness está descartado.

## Opções de correção

| Opção | Efeito | Custo |
|---|---|---|
| `<RuntimeIdentifiers>android-arm64</RuntimeIdentifiers>` (dropar 32-bit) | resolve de vez; correção de uma linha | perde devices 32-bit-only — o Play deixa de servir o app para eles. Não reduz o download de ninguém: com `androidpackageformat=aab` o Play já entrega apenas a slice da ABI do device |
| `RunAOTCompilation=false` | resolve em todas as ABIs | regressão de cold start para 100% da base |
| Quebrar `AsaasMobile.dll` em assemblies menores por feature | resolve mantendo AOT e 32-bit; o limite é por `.so` | refactor estrutural grande |
| Profile AOT customizado (`AndroidAotProfile`) reduzindo o conjunto de métodos AOT do app | resolve mantendo tudo | precisa gerar e manter o profile; investigar por que `androidenableprofiledaot=true` ainda produz 45 MB para o assembly do app |

Ordem de decisão recomendada:

1. Existe requisito de negócio para suportar `armeabi-v7a`? Se não, `android-arm64` é correção de uma linha.
2. Se existe, o caminho sustentável é reduzir o tamanho da imagem AOT do `AsaasMobile.dll` (quebra em assemblies ou profile customizado).
3. Desligar AOT globalmente é o pior trade: penaliza o startup de toda a base por causa de uma minoria de devices 32-bit.

## Prevenção

O `.text` do `libaot-AsaasMobile.dll.so` em `armeabi-v7a` é uma métrica de build que só sobe, e o custo de descobrir o estouro em produção é um crash 100% reproduzível no boot, sem stack trace gerenciado.

Vale um check que alerte (ou falhe) quando passar de ~30 MiB, usando o script da seção [Como diagnosticar](#como-diagnosticar-em-2-minutos). Ele precisa rodar **onde o AAB de release é de fato gerado** — não existe workflow de build Android neste repositório (`.github/workflows` tem apenas atlas-report, auto-bump-versions, merge-master-into-release, pull_request_toolkit, release-drafter e static-analysis), então o lugar é o pipeline externo de release.

**Automação:** ver `build-files/android/check_aot_text_size.py` — script de validação (sem dependências externas) que mede a seção `.text` de qualquer `.dll.so` recursivamente a partir de um caminho e falha o build se algum assembly 32-bit passar do limite configurado. Pensado para rodar como step do pipeline Azure DevOps logo após o build/publish Release Android, antes do empacotamento em AAB.
