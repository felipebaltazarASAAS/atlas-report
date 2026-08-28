# Migração AsaasMobile → AsaasMobile.Core

Guia para mover o conteúdo de `App/AsaasMobile` para `App/AsaasMobile.Core`, deixando
`AsaasMobile` como casca (entry points de plataforma, manifests, assinatura, ícone e splash).

## Desenho

| | `AsaasMobile` (casca) | `AsaasMobile.Core` (biblioteca) |
|---|---|---|
| Tipo | `OutputType=Exe` — app MAUI | biblioteca MAUI (sem `OutputType`) |
| `RootNamespace` | `AsaasMobile` (default) | `AsaasMobile` (explícito) |
| `AssemblyName` | `AsaasMobile` | `AsaasMobile.Core` |
| Conteúdo | `Platforms/`, manifests, ícone, splash, assinatura | todo o resto |

`RootNamespace` continua `AsaasMobile` no Core. Isso é o que faz a migração ser um
`git mv` puro:

- namespaces dos arquivos movidos não mudam (`AsaasMobile.Features.Login` continua igual);
- os nomes lógicos dos `.resx` continuam `AsaasMobile.Resources.Strings.*`, então os
  `*.Designer.cs` (que têm esse nome hardcoded) continuam funcionando sem alteração.

`TargetFrameworks`, `UseMaui`, `SingleProject`, `Nullable`, `ImplicitUsings`, os analyzers e os
`AdditionalFiles` vêm de `App/Directory.Build.props` — o Core não redeclara nada disso.

## Já feito

- `App/AsaasMobile.Core/AsaasMobile.Core.csproj` criado, buildando em `net9.0-android` e
  `net9.0-ios`.
- `PackageReference`s de feature (`BarcodeScanning.Native.Maui`, `Microsoft.Extensions.Logging.Debug`,
  `Otp.NET`, `PINView.MAUI`, `Plugin.Fingerprint_CalcDP_Maui`) movidos do `AsaasMobile.csproj`
  para o Core.
- `ProjectReference`s dos módulos compartilhados (Database, Framework, Gateway, GoogleWallet,
  Segment, Styleguide, Zendesk) movidos para o Core.
- `AsaasMobile.csproj` passou a referenciar o Core. Como `ProjectReference` e `PackageReference`
  são transitivos, a casca continua compilando com todo o código ainda dentro dela — dá para
  mover em partes, sem big bang.
- `AsaasMobile.Core.csproj` adicionado ao `AsaasMobile.slnx`.

## Fica na casca — não mover

Entry points e coisas que só funcionam no projeto de app:

- `Platforms/Android/MainActivity.cs`, `MainApplication.cs`, `DeepLinkActivity.cs`,
  `BankSlipOpenerActivity.cs`
- `Platforms/iOS/AppDelegate.cs`, `Program.cs`
- `Platforms/Android/AndroidManifest.xml` e `Platforms/iOS/Info.plist` — os workflows
  `.github/workflows/auto-bump-versions.yml` referenciam esses caminhos exatos
- Assinatura e ofuscação: `Platforms/Android/KeyStore/debug.keystore`,
  `Platforms/Android/proguard.cfg`,
  `Platforms/iOS/Entitlements-Dev.plist`, `Platforms/iOS/Entitlements-Prod.plist`
- `Platforms/Android/google-services.json`, `Platforms/iOS/GoogleService-Info.plist`,
  `Platforms/iOS/PrivacyInfo.xcprivacy`
- `Platforms/Android/Resources/**` (mipmaps, values, xml) e `Platforms/iOS/Assets.xcassets/**`
- `Resources/AppIcon/`, `Resources/Splash/` e os itens `MauiIcon` / `MauiSplashScreen`
- Os targets `_StripBitcodeFromFrameworks*` e `LinkWithSwift`, e todos os `PropertyGroup`
  condicionais de Android/iOS Debug/Release

## Move para o Core

- `Features/`, `Components/`, `Gateways/`, `Database/`, `Services/`, `Managers/`, `Trackers/`,
  `Converters/`, `Configurations/`, `FeatureControl/`, `Implementations/`, `Utils/`
- `App.cs`, `AsaasServices.cs`, `MauiProgram.cs`
- `Resources/Strings/`, `Resources/Styles/`, `Resources/Fonts/`, `Resources/Raw/`,
  `Resources/Animations/`, `Resources/Icons/`, `Resources/Images/`, `Resources/Shapes/`
- `Platforms/Android/Implementations/**`, `Platforms/iOS/Implementations/**`,
  `Platforms/iOS/Managers/**` — namespaces `AsaasMobile.Platforms.*`. Pastas
  `Platforms/{Android,iOS}` funcionam em biblioteca MAUI (o `Asaas.Framework` já faz isso).

## Passo a passo

### 1. Reescrever `assembly=AsaasMobile` nos XAMLs

Este é o único ajuste de código obrigatório. Há **1808 ocorrências em 829 arquivos** de
`;assembly=AsaasMobile` — depois do move, esse nome passa a apontar para a casca, que não tem
mais os tipos. Rodar **depois** de mover os XAMLs:

```bash
grep -rl 'assembly=AsaasMobile"' App/AsaasMobile.Core --include='*.xaml' \
  | xargs sed -i '' 's/assembly=AsaasMobile"/assembly=AsaasMobile.Core"/g'
```

Conferir que sobrou zero:

```bash
grep -rn 'assembly=AsaasMobile"' App/AsaasMobile.Core --include='*.xaml' | wc -l   # 0
```

Não confundir com `assembly=Asaas.Framework`, `assembly=Asaas.Styleguide` etc. — o padrão
`assembly=AsaasMobile"` (com a aspa) casa só com o alvo certo.

### 2. Mover `Features/` e `Configurations/` juntos

`Configurations/AsaasApplication.cs` faz:

```csharp
ViewModelMapper.Register(Assembly.GetExecutingAssembly(), "AsaasMobile.Features");
```

`GetExecutingAssembly()` é o assembly onde o código está. Se `AsaasApplication.cs` e `Features/`
ficarem em assemblies diferentes, o mapper registra o assembly errado e **nenhuma página é
encontrada** — sem erro de compilação, só `AsaasLogger.Warn` e tela em branco em runtime.

Se em algum momento parte das Features ficar na casca, é preciso uma segunda chamada de
`ViewModelMapper.Register` para o outro assembly.

Só tipos `public` são registrados (`assembly.ExportedTypes`) — o que já é o caso hoje.

### 3. Mover `Resources/Strings/` e transplantar os geradores

Levar do `AsaasMobile.csproj` para o `AsaasMobile.Core.csproj` o `ItemGroup` com os
`<EmbeddedResource Update="Resources\Strings\*.resx"><Generator>ResXFileCodeGenerator</Generator>`.

As classes geradas por `ResXFileCodeGenerator` são `internal`. Depois do move elas ficam visíveis
só dentro do `AsaasMobile.Core` — o que é suficiente, porque a casca não usa strings. Se algum
código que fica na casca precisar de uma string, trocar o gerador para
`PublicResXFileCodeGenerator`.

### 4. Mover `Resources/Icons`, `Images`, `Shapes`, `Fonts`, `Raw`, `Animations`

Levar junto os `ItemGroup` de `MauiImage`, `MauiFont` e `MauiAsset`.

`MauiImage` declarado no Core funciona — validado com um `MauiImage` de sonda: o build da
**biblioteca** não gera nada (`obj/.../res` vazio, `.aar` só com `proguard.txt`), quem processa é
o build do **app**, que produz as densidades em
`App/AsaasMobile/obj/Debug/net9.0-android/resizetizer/r/drawable-*/`. Ou seja: não estranhar o
`res` vazio do Core, e conferir sempre no output da casca:

```bash
ls App/AsaasMobile/obj/Debug/net9.0-android/resizetizer/r/drawable-xxhdpi/ | wc -l
```

`MauiIcon` e `MauiSplashScreen` continuam na casca — esses dois só funcionam no projeto de app.

`MauiFont` (`Resources\Fonts\*`) e `MauiAsset` (`Resources\Raw\**`,
`Resources\Animations\Jsons\**`) usam o mesmo mecanismo de coleta, mas **não foram validados** —
e, como imagens, são resolvidos por nome de arquivo em runtime. Se não propagarem, o sintoma é
fonte fallback (`AsaasStyleguide.RegisterFonts`) e animação Lottie faltando, sem nenhum erro de
build. Conferir depois do move:

```bash
# fontes e raw assets devem aparecer no output do app
find App/AsaasMobile/obj/Debug/net9.0-android -path '*assets*' -name '*.ttf' | head
find App/AsaasMobile/obj/Debug/net9.0-android -path '*assets*' -name '*.json' | head
```

### 5. Mover `MauiProgram.cs`, `App.cs`, `AsaasServices.cs`

Podem ir para o Core. `MainActivity`/`AppDelegate` continuam chamando
`MauiProgram.CreateMauiApp()` — o namespace `AsaasMobile` é o mesmo, resolve no Core sem
alteração de `using`.

## Armadilhas conhecidas

### NU1107 — `Xamarin.AndroidX.Fragment`

`BarcodeScanning.Native.Maui` traz `Xamarin.AndroidX.Fragment` por dois caminhos com ranges
incompatíveis:

- `→ Xamarin.Google.MLKit.BarcodeScanning → GooglePlayServices.Basement → Fragment >= 1.8.9.1`
- `→ Xamarin.AndroidX.Fragment.Ktx 1.8.6.1 → Fragment [1.8.6.1, 1.8.7)`

No `AsaasMobile` isso resolvia sozinho, no Core virou erro — com o **mesmo** conjunto de
`PackageReference` e `ProjectReference` (comparado nos dois `*.nuget.dgspec.json`). A única
diferença de input encontrada foi o nó `runtimes` que só o projeto de app tem (restore
RID-specific); não isolei o mecanismo exato. Resolvido com o pin que o próprio NuGet sugere, no
`AsaasMobile.Core.csproj`:

```xml
<ItemGroup Condition="$(TargetFramework.EndsWith('-android'))">
    <PackageReference Include="Xamarin.AndroidX.Fragment" Version="1.8.9.1" />
</ItemGroup>
```

`1.8.9.1` é a mesma versão que o `AsaasMobile` já resolvia antes da divisão — verificado no
`project.assets.json` antes e depois. Se aparecer NU1107 para outro pacote AndroidX ao mover mais
coisas, o padrão é o mesmo: pinar a versão que a casca já resolvia.

`Xamarin.Protobuf.JavaLite` continua **só** na casca — é dependência de empacotamento Java, não
é consumida por código C#, e o Core restaura sem ela.

### `Assembly.GetExecutingAssembly()`

Só existe uma ocorrência relevante fora do framework (`Configurations/AsaasApplication.cs`, ver
passo 2). O `Asaas.Framework` tem a sua própria em `AsaasFramework.cs`, que continua correta.
Se surgir código novo que dependa de "assembly que está rodando", lembrar que a partir daqui o
assembly de entrada é `AsaasMobile`, mas o assembly do código é `AsaasMobile.Core`.

## Ferramentas e CI que dependem do caminho

| Item | Situação |
|---|---|
| `.github/CODEOWNERS` (`*`) e `.github/labeler.yml` (`**`) | agnósticos de caminho, nada a fazer |
| `.github/workflows/auto-bump-versions.yml` | aponta para `AndroidManifest.xml` e `Info.plist`, que ficam na casca — nada a fazer |
| `.github/workflows/static-analysis.yml` (`cd App/AsaasMobile && dotnet restore`) | restaura o `.slnx`, que já inclui o Core — nada a fazer |
| `.github/scripts/generate-adoption-report.py` | **já ajustado**: resolve `Features/` em `AsaasMobile.Core` e cai para `AsaasMobile` se não existir, então funciona antes e depois do move |
| `build-files/android/check_aot_text_size.py` | usa `rglob("*.dll.so")` no diretório de AOT do app, então continua achando os `.so` — mas ver aviso abaixo |
| `scripts/find-unused-code.py` (não versionado) | tem `App/AsaasMobile` hardcoded como raiz do app; ajustar para `App/AsaasMobile.Core` ao mover |

### Aviso — guard de AOT `.text` (armeabi-v7a)

`check_aot_text_size.py` continua funcionando, mas **o arquivo que importa muda de nome**. Hoje
praticamente todo o código gerenciado AOTa em `libaot-AsaasMobile.dll.so`; depois do move vai para
`libaot-AsaasMobile.Core.dll.so`, e o `.so` da casca fica quase vazio.

Isso importa porque o limite de `.text` é **por arquivo `.so`** (ver
`docs/android-arm32-aot-plt-entry-crash.md`). Consequências:

- qualquer pipeline ou análise que olhe `libaot-AsaasMobile.dll.so` pelo nome passa a medir um
  arquivo vazio e a aprovar silenciosamente;
- a divisão em si **não reduz** o `.text` total — só troca o nome do arquivo onde ele mora.

O guard hoje reprova pela condição exata da falha (PLT além do alcance do `BL`, ou veneers apontando
para ela), não só por tamanho; a saída traz a posição da PLT e a margem de cada imagem.

Rodar o guard explicitamente depois do move, num build Release Android com AOT, e conferir que ele
lista o `.so` do Core:

```bash
python3 build-files/android/check_aot_text_size.py \
    App/AsaasMobile/obj/Release/net9.0-android/android-arm/aot
```

## Verificação

```bash
# restore das duas pontas
dotnet restore App/AsaasMobile.Core/AsaasMobile.Core.csproj
dotnet restore App/AsaasMobile/AsaasMobile.csproj

# build por plataforma
dotnet build App/AsaasMobile/AsaasMobile.csproj -f net9.0-android -c Debug
dotnet build App/AsaasMobile/AsaasMobile.csproj -f net9.0-ios -c Debug
```

Checar que a versão resolvida dos pacotes AndroidX não mudou:

```bash
grep -o '"Xamarin.AndroidX.Fragment/[0-9.]*"' App/AsaasMobile/obj/project.assets.json | sort -u
# "Xamarin.AndroidX.Fragment/1.8.9.1"
```

Build Debug compilando não é suficiente para dar a migração como concluída. Antes de abrir PR:

- rodar o app em Android e iOS e navegar por pelo menos uma tela de cada área — quebra de
  `ViewModelMapper` (passo 2) não aparece em build;
- conferir que ícones e imagens aparecem (passo 4);
- build `Release` de Android (`r8` + trimming) — é o caminho que mais muda com a troca de
  assembly de entrada.
