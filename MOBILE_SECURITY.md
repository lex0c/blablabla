# Segurança Mobile

Apps mobile enfrentam um modelo de ameaça particular: o binário é distribuído ao atacante; o dispositivo pode estar rooted/jailbroken; o código roda em ambiente parcialmente confiável. Tudo que é "segredo do cliente" pode ser extraído.

## Modelo Android

- **Linux kernel** + Android Runtime (ART).
- **App sandbox**: cada app em UID próprio; IPC via Binder (intents, content providers, services).
- **Permissions**: install-time (legacy, obsoleto) e runtime (Android 6+). Permissions sensíveis exigem prompt.
- **Scoped storage** (Android 10+): apps não leem arbitrariamente storage externo.
- **SELinux** em enforcing mode.
- **Keystore**: chaves em hardware-backed (TEE/StrongBox) quando disponível.
- **Play Protect**: varredura de apps.
- **Verified boot**: cadeia de assinatura desde bootloader.

### Vetores comuns

- **WebView** com `setJavaScriptEnabled(true)` + `addJavascriptInterface` → XSS na webview leva a RCE.
- **Intents exportados** sem verificação → outros apps invocam.
- **Content providers** com `android:exported="true"` sem permission.
- **Deep links** sem validação → phishing, abuse de confiança.
- **Backup habilitado** (`android:allowBackup="true"`): ADB backup de dados do app.
- **Debuggable** (`android:debuggable="true"`): nunca em produção.
- **SSL pinning quebrado** → MITM de tráfego HTTPS.
- **Armazenamento inseguro**: SharedPreferences em plaintext, SQLite sem encryption, logs com PII.

## Modelo iOS

- **Darwin/XNU kernel**.
- **Sandbox profiles** por app (baseado em seatbelt/sandbox\_init).
- **Entitlements**: permissões em cert do app.
- **Code signing** obrigatório; binários só rodam assinados.
- **Data Protection**: arquivos criptografados com chave derivada do passcode; classes (complete, complete until unlock, etc).
- **Keychain**: storage protegido hardware-backed.
- **App Transport Security (ATS)**: exige HTTPS por default.
- **Pointer Authentication (PAC)** em ARMv8.3+: mitiga ROP.

### Vetores comuns

- **URL schemes** sem validação → abuse.
- **Universal Links** mal configurados → app shim.
- **NSAllowsArbitraryLoads** desabilitando ATS.
- **WKWebView + JS bridges** mal desenhados.
- **Clipboard** leaks (iOS 14+ avisa, ajudou muito).
- **Keyboard extensions** com acesso full → captura.

## Ferramentas de Análise

### Estática

- **apktool**: decodifica APK → smali.
- **jadx**: decompila DEX para Java.
- **MobSF (Mobile Security Framework)**: análise automatizada estática+dinâmica, excelente ponto de partida.
- **objection** (iOS/Android): runtime mobile exploration.
- **class-dump** (iOS): classes Obj-C.
- **Ghidra, IDA**: para `.so` nativas e binários iOS/arm64.

### Dinâmica

- **Frida**: injeta JS em processo mobile. Hooks em métodos Java/Swift/Obj-C/native. Padrão de facto em análise dinâmica mobile.
  - Exemplos: `Java.use("com.app.Auth").isRooted.implementation = function() { return false; }` — bypass instantâneo de root detection.
- **Objection**: wrapper de Frida com comandos úteis.
- **Burp Suite / mitmproxy** com CA instalada no device: intercepta HTTPS.
- **Charles Proxy**.

## Certificate Pinning e Bypass

Apps de alto valor (bancos, wallet) fazem pinning — validam contra cert específico, não só CA confiável. Bypass de pinning é o primeiro passo em análise de rede de apps sérios.

- **Frida scripts**: `frida-multiple-unpinning`, `universal-android-ssl-pinning-bypass`. Funcionam em ~80% dos apps.
- **Apps com pinning customizado**: exige hook manual da função de validação.
- **Apps com detecção de Frida**: jogo de gato e rato (anti-debug, integrity check, SafetyNet/Play Integrity).

**Para o defensor**: pin com SPKI, múltiplos pins (cert + backup), fallback controlado.

## Root / Jailbreak Detection

Apps detectam ambientes comprometidos para recusar funcionar. Técnicas típicas:

- Presença de `su`, `/system/xbin/su`, `Superuser.apk`, Magisk.
- Writeable `/system`.
- Instalação de apps conhecidos (Cydia, Magisk Manager).
- `RootBeer`, `SafetyNet Attestation`, `Play Integrity` (Android); `DTCheckRoot` (iOS).

Bypass:

- **Magisk Hide / DenyList** oculta do app.
- **Frida hook** na função que decide.
- **Shamiko, LSPosed** para fine-grained.

É **cat-and-mouse**: detecção perfeita é impossível quando atacante controla o dispositivo.

## SafetyNet / Play Integrity (Android)

Serviço do Google que atesta ao servidor que o device é genuíno, ROM de fábrica, etc. Verificação ocorre no servidor com token emitido pelo Google. Importante: **verificar no backend, não no app** — senão o hook mente.

iOS equivalente: **DeviceCheck** + **App Attest**.

## Armazenamento Seguro

### Android

- **EncryptedSharedPreferences** (Jetpack Security).
- **EncryptedFile** para arquivos.
- **Keystore** para chaves privadas (biometria unlock, hardware-backed).
- **Room com SQLCipher** para DB criptografado.

### iOS

- **Keychain** com `kSecAttrAccessible*` apropriado.
- **Data Protection API** com `NSFileProtectionComplete` quando possível.
- **Core Data / Realm com encryption**.

Nunca em:
- Plain `SharedPreferences` / `NSUserDefaults`.
- `/sdcard/` (Android).
- Logs.

## Autenticação em Mobile

- **Biometria**: `BiometricPrompt` (Android), `LocalAuthentication` (iOS). Combinar com Keychain/Keystore — biometria desbloqueia chave, não retorna "true/false".
- **Token storage**: Keychain/Keystore; não SharedPreferences.
- **Session timeout** aparente ao usuário; revogação server-side.
- **MFA** em escalação crítica.

## API Security

O backend mobile frequentemente é mais vulnerável que o app em si.

- **Client-side checks are advisory**. Tudo se repete no servidor.
- **Rate limiting** por usuário + device.
- **Integrity attestation** (Play Integrity, App Attest) para gating, não autenticação.
- **API keys não são segredos em apps**. Qualquer atacante extrai. Use tokens efêmeros com OAuth.

## Distribuição

- **Play Store, App Store**: assinatura, review, capacidade de revogar.
- **APK sideload** (Android): qualquer apk fora do Play; atacante redistribui app original com trojan.
- **Certificate revocation**: iOS revoga enterprise certs usados abusivamente.

## Testes

- **MSTG / MASVS** (OWASP Mobile Application Security): checklist + verification standard.
- **MASA** (Mobile App Security Assessment): certificação Google.
- **Frida, objection, MobSF** — já citados.
- **drozer** (Android): framework para fuzzing de componentes exportados.
- **iOS**: precisa de device jailbroken para profundidade real; `checkra1n`, `palera1n`, `unc0ver` variam com versão iOS.

## Ferramentas de RE Mobile

Consolidando:

- **Android**: `apktool`, `jadx`, `dex2jar`, `smali/baksmali`, `frida`, `objection`, `MobSF`.
- **iOS**: `Ghidra`, `Hopper`, `IDA`, `class-dump`, `Cycript`, `Frida`, `iproxy`, `fridump`.

## Exemplos de CVEs Mobile Famosos

- **Stagefright** (Android, 2015): parsing de MMS → RCE sem interação.
- **BlueBorne**: Bluetooth stack.
- **CVE-2019-2215** (Binder UAF): kernel LPE.
- **iOS imessage zero-click chains** (2020-2022): Pegasus.
- **Dirty COW / Dirty Pipe** também afetam Android.

## Pegasus e Commercial Surveillance

Mercenary spyware (NSO Group, Intellexa). Exploits zero-click exorbitantes vendidos a governos. Alvos: jornalistas, ativistas, políticos.

- **iOS Lockdown Mode**: perfil restritivo anti-spyware. Usar se alto risco.
- **Relatórios**: Citizen Lab, Amnesty Security Lab.

## Considerações de Privacidade

- **Permissions**: pedir só o essencial; justificar na UI.
- **Telemetry**: minimizar, anonimizar. LGPD/GDPR aplicam.
- **Third-party SDKs**: cada um é um risco de privacidade. Auditar.
- **Tracker scanners**: ExodusPrivacy.

## Princípios

1. **Cliente é zona hostil**. Nada sensível é "secreto" no app.
2. **Servidor decide autorização**. Cliente só apresenta; backend valida.
3. **Storage cifrado por default** em chaves/segredos.
4. **Pinning com fallback**: pin rígido quebra na rotação.
5. **Integrity attestation** em backend, não em client.
6. **Menos permissões = mais confiança** do usuário e menor blast radius.
7. **Atualize dependências**: libs mobile com CVEs vivem no seu app até o próximo release.
