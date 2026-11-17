# Anti-Hook Hardening — Test Scenarios & PoC Scripts

Documentação completa de todos os cenários de teste para validação do hardening contra Frida, root detection bypass, e tampering detection.

---

> **Escopo desta branch (MBT-5347):** só os cenários **C3** (detecção de Frida) e **C6** (ofuscação de keywords) se aplicam. Os cenários **C1/C2** (guard nativo, DenyList, nonce) e **C4** (integração `IsSecureDevice` com `NativeGuard`) testam código que **não está nesta branch** e não devem ser executados aqui.
>
> **Pré-requisito para qualquer cenário que espere o app encerrar:** o encerramento é gateado por um campo do `AppInitializationConfigDTO`, desligado por padrão. Para testar o encerramento, ligue o campo (ambiente de teste ou build local); com o gate desligado o app não encerra e apenas registra. Use o pacote real `asaas.asaas` (activity `crc645b4101823ca26ade.MainActivity`). Leitura de `/proc` deve usar `/proc/$(pidof asaas.asaas)/...`, e não `/proc/self` do shell. O exemplo de C6 abaixo é ilustrativo (pseudocódigo) e não reproduz a implementação de `ObfuscatedKeyword.cs`.

## 📋 Visão Geral dos Testes

| Camada | Implementação | PoC Script | Status | Objetivo |
|--------|---|---|---|---|
| C1/C2 | Root Detection Anti-DenyList | `c1_test.py` | ✓ | Bypass root detection com DenyList |
| C2 | Static Hook Nonce Validation | `c2_test.py` | ✓ | Bypass nonce validation |
| C3 | Frida Detection Multi-Vetor | `c3_test.py` | ✓ | Bypass detecção multi-vetor |
| C4 | Defense-in-Depth Integration | `c4_test.py` | ✓ | Bypass integração fail-closed |
| C6 | Keyword Obfuscation | `c6_test.py` | ✓ | Extrair keywords obfuscadas |

---

## 🧪 Cenários de Teste Detalhados

### C1/C2 — Root Detection Anti-DenyList

**Localização:** `/workspace/asaas/frida/c1_test.py`

**Problema Abordado:**
```
Magisk DenyList esconde root → RootBeer vê "não rootado" (falso negativo)
Solução: Detecção nativa usando behavioral signals
```

**Cenários:**

#### C1-S1: Normal app launch (sem hooks)
- **Setup:** App limpo, sem Magisk DenyList
- **Esperado:** App abre normalmente, sem bloqueios
- **Validação:** Confirmar visualmente que app está funcional

#### C1-S2: Hide root com Magisk DenyList
- **Setup:** `magisk --denylist add asaas.asaas`
- **Esperado:** App deteta root mesmo com DenyList
- **Validação:** App mostra DeviceBlockedPage (C1/C2 working)
- **Bypass Tentado:** RootBeer checks + behavioral signals não são fooled

#### C1-S3: Check behavioral signals
- **Signals Verificados:**
  - TracerPid != 0 em `/proc/self/status`
  - Mount namespaces alterados
  - Binaries substituídos
  - Nonce validation failures
- **Esperado:** Nativo consegue ver behavioral signals mesmo com DenyList
- **Validação:** Detecção nativa dispara no startup

**Comandos Úteis:**
```bash
# Verificar TracerPid
adb shell cat /proc/self/status | grep TracerPid

# Aplicar DenyList
adb shell magisk --denylist add asaas.asaas

# Remover DenyList
adb shell magisk --denylist remove asaas.asaas

# Reiniciar app
adb shell am start -n asaas.asaas/crc645b4101823ca26ade.MainActivity
```

---

### C2 — Static Hook Nonce Validation

**Localização:** `/workspace/asaas/frida/c2_test.py`

**Problema Abordado:**
```
Atacante hookeava detecção nativa → app via como "safe"
Solução: Nonce validation contra static hooks
```

**Cenários:**

#### C2-S1: Normal startup (sem Frida)
- **Setup:** App sem hooks, sem Frida
- **Esperado:** App abre normalmente
- **Validação:** MainActivity visible

#### C2-S2: Attempt to hook ValidateNonce
- **Setup:** Frida server running, script hooks ValidateNonce
- **Esperado:** Hook é detectado e app bloqueado
- **Validação:** DeviceBlockedPage (static hook detected)
- **Bypass Tentado:** Forçar return true na validação

#### C2-S3: Multiple simultaneous hooks
- **Setup:** Frida hookeando N funções de segurança
- **Esperado:** Presença de múltiplos hooks = Frida detection
- **Validação:** App bloqueado
- **Bypass Tentado:** Esconder hooks com Frida internals

**Comandos Úteis:**
```bash
# Iniciar Frida server
adb shell /data/local/tmp/frida-server &

# Listar processos
frida-ps -U

# Rodar script
frida -U asaas.asaas -l hook_script.js

# Check nonce validation
adb logcat | grep -i "nonce\|validation"
```

---

### C3 — Frida Detection Multi-Vetor

**Localização:** `/workspace/asaas/frida/c3_test.py`

**Problema Abordado:**
```
Port 27042 detectado mas não matava → looper.Cancel() paralisava checks
Solução: Multi-vetor detection + no cancel (C4)
```

**Cenários:**

#### C3-S1: Check Frida indicators (port, memfd, WebSocket)
- **Indicators Verificados:**
  - Port 27042 open
  - `/memfd:frida-agent` em `/proc/self/maps`
  - WebSocket handshake response (GET /ws → 101)
  - Stacktraces com ptrace, mmap, substrate
  - TracerPid != 0

- **Esperado:** Qualquer um desses é suficiente para bloqueio
- **Validação:** App bloqueado ao detectar qualquer indicador

#### C3-S2: Normal app startup (sem Frida)
- **Setup:** App sem Frida running
- **Esperado:** Nenhum indicador encontrado, app funciona
- **Validação:** MainActivity open

#### C3-S3: Frida server running
- **Setup:** `adb shell /data/local/tmp/frida-server &`
- **Esperado:** App detecta port 27042 e bloqueia
- **Validação:** DeviceBlockedPage
- **Bypass Tentado:** Conectar em porta alternativa, hide memfd

#### C3-S4: Late-attach Frida
- **Setup:** App already open, depois `frida -U app`
- **Esperado (sem C4):** Gap entre startup check e attach = bypass
- **Esperado (com C4):** Continuous checks detectam late-attach
- **Validação:** App bloqueado mesmo após late-attach

**Comandos Úteis:**
```bash
# Verificar port 27042
adb shell netstat -tuln | grep 27042

# Verificar memfd
adb shell cat /proc/self/maps | grep memfd

# Ver stacktrace indicators
adb shell logcat | grep -i "ptrace\|mmap\|substrate"

# Rodar Frida
frida -U asaas.asaas
```

---

### C4 — Defense-in-Depth Integration (IsSecureDevice)

**Localização:** `/workspace/asaas/frida/c4_test.py`

**Problema Abordado:**
```
Looper se auto-cancelava ao primeiro flag → late-attach não era detectado
Solução: IsSecureDevice integra ALL vetores + no cancel
```

**Cenários:**

#### C4-S1: Clean startup
- **Setup:** App sem hooks
- **Esperado:** IsSecureDevice() → true, app works
- **Validação:** App aberto normalmente

#### C4-S2: Hook IsSecureDevice directly
- **Setup:** Frida hooks IsSecureDevice() → return true
- **Esperado:** Hook é detectado, app bloqueado
- **Validação:** DeviceBlockedPage
- **Razão:** Presença de Frida para fazer o hook é detectada por C3

#### C4-S3: Hook individual checks (C1/C2, C3)
- **Setup:** Frida hooka:
  - NativeGuard.IsTampered()
  - FridaDetectionUtils.CheckFridaPresence()
  - HookDetectionUtils.CheckSuspiciousIndicators()
- **Esperado:** Múltiplos hooks = Frida presence → block
- **Validação:** DeviceBlockedPage
- **Razão:** C4 checa todos; falha em qualquer um bloqueia

#### C4-S4: Late-attach after app initialization
- **Setup:**
  1. App abre (clean)
  2. User navega para dentro
  3. Frida late-attach
- **Esperado (sem C4):** No detectable gap
- **Esperado (com C4):** Continuous checks disparam
- **Validação:** App detecta late-attach e bloqueia
- **Razão:** IsSecureDevice não tem cancel; continuous checks

#### C4-S5: Race condition between checks
- **Setup:** Frida hookeando N functions simultaneous
- **Esperado:** Mesmo com hooks parciais, first failure bloqueia
- **Validação:** Early return → fast block
- **Razão:** C4 usa early return, não OR/AND chains

**Fluxo C4:**
```csharp
IsSecureDevice() {
    if (NativeGuard.IsTampered()) return false;        // C1/C2 — falha → bloqueia
    if (HasSuspectFileInMemory()) return false;        // C3 — falha → bloqueia
    if (HasSuspectOpenPorts()) return false;           // C3 — falha → bloqueia
    return true;                                        // Todos passam
}
```

**Comandos Úteis:**
```bash
# Monitor IsSecureDevice calls
adb logcat | grep -i "issecuredevice\|account.*security"

# Check device blocked page
adb logcat | grep -i "blocked\|access.*denied"

# Late-attach test
frida -U asaas.asaas --raw
```

---

### C6 — XOR-Encoded Keyword Obfuscation

**Localização:** `/workspace/asaas/frida/c6_test.py`

**Problema Abordado:**
```
Keywords plaintext ("frida", "ptrace") → string search no APK → bypass automático
Solução: Keywords XOR-encoded, decodificadas em runtime
```

**Cenários:**

#### C6-S1: Static analysis (APK strings)
- **Setup:** Extrai strings da APK
- **Esperado:** Nenhum plaintext keyword
- **Validação:** grep "frida\|ptrace\|mmap\|substrate" retorna 0
- **Comando:** `strings app.apk | grep -i frida`

#### C6-S2: APK decompilation
- **Setup:** Decompila APK com apktool
- **Esperado:** ObfuscatedKeyword class with XOR logic
- **Validação:** Encoding logic não trivial de reverter
- **Bypass Tentado:** Encontrar XOR key ou valores decodificados

#### C6-S3: Runtime extraction
- **Setup:** Frida hookeando Decode() method
- **Esperado (sem C6):** Keywords visíveis via inspection
- **Esperado (com C6):** Decodificação em runtime, extraction difícil
- **Validação:** Não consegue extrair keywords pré-decodificadas
- **Razão:** Keywords apenas em memory, nunca em plaintext

#### C6-S4: Dynamic mutation (C7 - Future)
- **Setup (v2.137.0+):** XOR key randomizada por build
- **Esperado:** Cada APK tem keywords diferentes
- **Validação:** mesmo source code → diferentes binaries
- **Benefício:** Extraction em uma build não funciona em outra

**Encodificação C6:**
```csharp
// Encoded string (example)
private static readonly byte[] FRIDA_KEYWORD_ENCODED = new byte[] { 0x8F, 0x9D, 0x8A, 0x85, 0x90 };
private static readonly byte XOR_KEY = 0x42;

public static string Decode(byte[] encoded) {
    byte[] decoded = new byte[encoded.Length];
    for (int i = 0; i < encoded.Length; i++) {
        decoded[i] = (byte)(encoded[i] ^ XOR_KEY);
    }
    return Encoding.ASCII.GetString(decoded);
}
```

**Comandos Úteis:**
```bash
# Extrair strings
strings app.apk | grep -i "frida\|ptrace"

# Decompile
apktool d app.apk -o app_decompiled

# Procurar encoding
grep -r "ObfuscatedKeyword\|XOR\|Decode" app_decompiled

# Runtime extraction com Frida
frida -U asaas.asaas -l extract.js
```

---

## 🚀 Como Executar Testes

### Pré-requisitos

```bash
# Device setup
adb connect <device>

# App installed
adb install app-release.apk

# Magisk (para C1 tests)
adb shell magisk --version

# Frida
adb shell /data/local/tmp/frida-server &
frida --version

# Python dependencies
pip install frida-tools
```

### Executar todos os testes

```bash
cd /workspace/asaas/frida/
chmod +x run_all_tests.sh
./run_all_tests.sh
```

### Executar teste específico

```bash
cd /workspace/asaas/frida/

# C1/C2 — Root Detection
python3 c1_test.py

# C2 — Static Hooks
python3 c2_test.py

# C3 — Frida Detection
python3 c3_test.py

# C4 — Defense-in-Depth
python3 c4_test.py

# C6 — Keyword Obfuscation
python3 c6_test.py
```

---

## 📊 Matriz de Teste

| Vetor | C1 | C2 | C3 | C4 | C6 |
|-------|----|----|----|----|-----|
| Magisk DenyList | ✓ | - | - | - | - |
| Static hooks | ✓ | ✓ | - | ✓ | - |
| Frida port | - | - | ✓ | ✓ | - |
| memfd escape | - | - | ✓ | ✓ | - |
| Late-attach | - | - | ✓ | ✓ | - |
| String search | - | - | - | - | ✓ |
| APK decompile | - | - | - | - | ✓ |

---

## 📝 Relatório de Teste

Após executar os testes, documentar:

```markdown
## Test Results

**Date:** YYYY-MM-DD
**Tester:** [name]
**Device:** [model, Android version]
**Build:** [version, debug/release]

### C1/C2 — Root Detection
- [ ] Normal startup: PASS / FAIL
- [ ] DenyList bypass: PASS / FAIL (app blocked)
- [ ] Behavioral signals: PASS / FAIL (detected)

### C2 — Static Hooks
- [ ] No hooks: PASS / FAIL
- [ ] Nonce hook: PASS / FAIL (app blocked)
- [ ] Multiple hooks: PASS / FAIL (app blocked)

### C3 — Frida Detection
- [ ] No Frida: PASS / FAIL
- [ ] Port 27042: PASS / FAIL (detected)
- [ ] memfd maps: PASS / FAIL (detected)
- [ ] Late-attach: PASS / FAIL (detected)

### C4 — Defense-in-Depth
- [ ] Clean startup: PASS / FAIL
- [ ] Hook IsSecureDevice: PASS / FAIL (blocked)
- [ ] Hook all checks: PASS / FAIL (blocked)
- [ ] Late-attach: PASS / FAIL (blocked)

### C6 — Keyword Obfuscation
- [ ] String search: PASS / FAIL (no keywords)
- [ ] Decompilation: PASS / FAIL (XOR logic found)
- [ ] Runtime extraction: PASS / FAIL (cannot extract)

### Overall Result
- **Status:** ALL PASS / SOME FAIL / ALL FAIL
- **Gaps:** [List any gaps found]
- **Notes:** [Additional observations]
```

---

## 🔮 Roadmap Futuro

### v2.137.0+ — Dynamic Randomization

- **C5:** Port randomized per build (26000-28000)
- **C7:** XOR key randomized per build
- **Benefit:** Extraction on one build ≠ applicable to other builds

### Future F1 — Kernel-Level Detection

- Behavioral detection at kernel level
- Monitor system calls (ptrace, mmap, etc.)
- Tamper-proof logging
- Detection invisible to userland Frida

---

## 📚 Referências

- **PoC Bypass:** `ANTI-HOOK-BYPASS.md` (mantido fora do repositório, junto da instrumentação)
- **Implementation Plan:** [ANTI_HOOK_HARDENING.md](ANTI_HOOK_HARDENING.md)
- **Corrections:** `anti-hook-corrections-plan.md` (planejado, ainda não publicado)
- **Protection Guide:** `anti-hook-protection-plan.md` (planejado, ainda não publicado)

---

**Created:** 2026-06-10  
**Last Updated:** 2026-06-10  
**Status:** Ready for device testing
