...

## IOC / IOA

### **IOC: pegadas do que já aconteceu**

* São **evidências concretas** de que um dispositivo foi comprometido.
* Em mobile, podem ser:

  * Hash de um APK malicioso.
  * Domínio/IP de C2 contatado por um trojan bancário.
  * Arquivos estranhos criados no `/sdcard/Android/data/.hidden`.
  * Permissões de acessibilidade ativadas sem relação com a função do app.
* Valor: permitem bloquear e caçar *algo que já conhecemos*, mas só **depois** do ataque já estar em andamento.
* Limitação: atacantes trocam IOC fácil (novo hash, novo domínio).

### **IOA: sinais de que o ataque está acontecendo**

* São **padrões de comportamento** que revelam intenção maliciosa antes do dano final.
* Em mobile, exemplos típicos:

  * Uso de `DexClassLoader` para carregar código de um blob criptografado em runtime.
  * App de lanterna pedindo `BIND_ACCESSIBILITY_SERVICE`.
  * Foreground service “fantasma” que mantém processo vivo com notificação genérica.
  * Criação de dezenas de *BroadcastReceivers* para se manter ativo após boot, rede ou tela ligar.
* Valor: permitem **interromper** ataques mesmo que o hash/domínio do malware seja novo e desconhecido.
* Limitação: exigem contexto (nem todo uso de `DexClassLoader` é malicioso), ou seja, precisam de análise dinâmica e correlação.

---

## Técnicas clássicas de **sandbox evasion**

- **Checagem de sensores “irreais”**

   * GPS parado, sempre na mesma coordenada.
   * Acelerômetro sem movimento.
   * Câmera ou microfone inexistente.
   * Se não há variabilidade, o app assume “isso é uma VM/sandbox”.

- **Bateria e energia**

   * Emuladores normalmente têm bateria travada em 100% ou sem desgaste.
   * Malware pode simular carga e esperar mudanças. Sem mudança = suspeito.

- **Ambiente de rede**

   * Sandbox muitas vezes só permite conexões limitadas ou via proxy.
   * Se o IP é de ranges conhecidos da Google, Amazon ou serviços de análise, o malware fica quieto.

- **Checagem de hardware**

   * Quantidade de RAM absurda ou baixa demais.
   * CPU virtual (QEMU, Intel VT-x).
   * Dispositivos USB e sensores ausentes.
   * Nomes estranhos em build.prop (Android) tipo `generic_x86`.

- **Tempo de uso do aparelho**

   * Emuladores são “novos em folha”: uptime curto, lista de contatos vazia, nenhum histórico de chamadas, galeria limpa.
   * Se não há sinais de vida real, o malware desliga o modo maligno.

- **Comportamento atrasado**

   * Dormir horas, dias ou só ativar em horários aleatórios.
   * Força os analistas a manter o sandbox rodando por muito mais tempo (caro e inviável em escala).

- **Checagem de apps instalados**

   * Sandbox geralmente tem só o app testado e alguns utilitários.
   * Se não há WhatsApp, Facebook, Gmail etc., suspeito.
   * Malware pode esperar até detectar um “ecossistema humano real” antes de agir.

- **Uso de triggers externos**

   * Só ativa ao receber SMS, push ou comando remoto do C2.
   * Sandbox dificilmente simula todos esses canais.

---

## Como pegar apps que tentam se esconder

* Varie sensores de forma crível: GPS com jitter sutil, acelerômetro com ruído, passos simulados, mudança de orientação, fotos falsas na galeria, contatos e histórico mínimos porém plausíveis.
* Perfil de bateria realista: estado flutuando entre 20% e 87%, ciclos de carga, temperatura variando pouco.
* Build fingerprint e props: nada de `generic_x86`. Use fingerprints de dispositivos reais rotacionados e coerentes com a versão do Android, segurança e patch level.
* Rede parecida com usuário final: ASN residencial ou móvel, DNS realista, latências variáveis, horários de uso humanos. Bloqueie detecção fácil de proxy.
* Uptime e “idade” do device: simule tempo desde o boot e desde a ativação, apps populares instalados, caches, notificações antigas.
* Gatilhos externos: injete SMS, push, mudanças de conectividade, alarme do sistema e eventos de boot para forçar caminhos de execução.
* Execução prolongada e aleatória: rode por dias com janelas de atividade randômicas. Malwares dormem para driblar análises de curto prazo.
* Instrumentação agressiva: hook de APIs sensíveis, log de chamadas nativas e reflexão, inspeção de decriptação de payload em runtime, snapshots de memória quando o processo tocar em crypto ou net.
* Análise diferencial: compare comportamento do mesmo binário em ambiente “óbvio” vs ambiente “realista”. Divergência forte costuma indicar evasão.
* Monitoramento pós-publicação: telemetria de egress suspeito, spikes de permissões raras, foreground service persistente disfarçado de player ou VPN, broadcast receivers exagerados.

### Sinais de alerta no APK sem precisar de “código de evasão”

* Uso pesado de reflection e `DexClassLoader`, blobs AES que viram classes em runtime, verificação obsessiva de build props e sensores, timers longos sem propósito funcional, recepção de muitos broadcasts do sistema para persistência.

### Estratégia de resposta

* Gate por feature: limite em produção o acesso a APIs críticas até o app passar por um canário com ambiente realista.
* Kill switch remoto: prepare política para revogar versões e bloquear C2 detectado via IOC.
* Reassinar e reempacotar em laboratório: se o app verifica assinatura, isso revela lógica antitamper.



## Sandbox: Arquitetura em 6 blocos

1. Orquestração de jobs

   * Kubernetes ou Nomad. Um pod por amostra. Timeout longo com janelas de atividade randômicas.

2. Dispositivo analisador

   * Emulador AOSP ARM com imagem userdebug e sistema gravável. Frida server no device. Cert do proxy instalado no keystore do sistema.

3. Realismo de usuário

   * Seeder carrega contatos mínimos, 30 fotos fake, histórico de chamadas curto, 8 apps populares, calendário com eventos. Script de bateria e movimento leves.

4. Rede observável

   * mitmproxy para captura de HTTP e TLS onde possível, pcap no host, DNS resolver próprio com logging. Saída pela ASN residencial ou móvel via eSIM ou tunnel.

5. Instrumentação

   * Frida para hooks em APIs de rede, crypto, job scheduling, dynamic loading, localização. Syscall trace para file I/O suspeito.

6. Coleta e análise

   * Eventos para Kafka. Normalização em stream. Persistência em Postgres para metadados, S3 para pcaps e dumps. Dashboard em Grafana, alertas em Slack.

### Realismo do device

* Build props coerentes com modelo e patch level. Sem `generic_x86`.
* Bateria com variação entre 23 e 81 por cento, temperatura e voltagem flutuando pouco.
* Sensores com jitter baixo. Passos, rotação de tela, unlocks ocasionais.
* Uptime e idade. Primeiro boot “uma semana atrás”, não 5 minutos.
* Agenda de uso humano. Picos 12h e 20h, silêncio de madrugada.

### Instrumentação recomendada

Colete sem tentar “quebrar” nada. Só observe.

* Rede

  * `java.net.HttpURLConnection`, OkHttp, Cronet, WebView, SocketChannel. Log de host, SNI, porta, bytes, user agent.

* Crypto e codecs

  * `Cipher.doFinal`, `MessageDigest.digest`, `Base64.decode`. Hash do input e tamanho, nunca o plaintext bruto se contiver PII.

* Persistência e execução

  * `dalvik.system.DexClassLoader`, `PathClassLoader`, `Runtime.loadLibrary`, `ProcessBuilder`. Log de caminhos e hashes.

* Agendadores e persistência

  * `JobScheduler.schedule`, `AlarmManager.setExact`, `WorkManager.enqueue`. Frequência e flags.
  * `BroadcastReceiver` registrados em runtime para `BOOT_COMPLETED`, `CONNECTIVITY_CHANGE`.

* Privacidade e sensores

  * `TelephonyManager`, `AccessibilityService`, `DevicePolicyManager`. Permissões solicitadas e uso real.

### Gatilhos e rotina do experimento

* Instala e abre o app via `adb` com perfis de usuário distintos.
* Janelas de uso ativas por 5 a 15 minutos, 3 a 6 vezes por dia.
* Dispara broadcasts comuns

  * `BOOT_COMPLETED`, alterna Wi-Fi e dados, altera zona, muda idioma, pluga e desplug a energia, envia notificação fake.
* Injeta dados realistas

  * 3 SMS recebidos, 2 chamadas perdidas, 4 notificações de chat.
* Observa 48 horas. Amostras que “dormem” mais ganham outra janela programada.

### Heurísticas e score

* EvasionScore

  * Leitura agressiva de build props, sensores sem função visível, divergência de comportamento entre perfil “óbvio” e “realista”, timers longos sem UX correspondente.
* PersistenceScore

  * Registro de 5+ receivers de sistema, uso repetido de `setExactAndAllowWhileIdle`, foreground service disfarçado com notificação genérica.
* C2Score

  * Domínios curtos e rotativos, resolução com TTL anêmico, beacons periódicos disfarçados de analytics, JARM ou JA3 raros, HTTP2 com caminhos randômicos.
* StealthScore

  * `DexClassLoader` em assets criptografados, execução só após janelas de inatividade, code loading por WebView.

Produza um `RiskScore = f(Evasion, Persistence, C2, Stealth)` com pesos. Gere IOC

* Domínios, IPs, JA3, caminhos, hashes de payload dinâmico, nomes de serviços e receivers registrados.

### Relatórios e saída

* Timeline de eventos com marcadores de gatilhos.
* Grafo de rede por host e volume.
* Trilha de scheduling e reinícios de serviço.
* Pacote IOC assinável para SIEM.

### Operação e higiene

* Use imagens efêmeras. Limpe o device entre amostras.
* Sem contas reais. Dados seed sempre sintéticos.
* Rate limit e isolamento forte de rede.
* Registro imutável dos resultados para cadeias de custódia.


---
...

### 1. **Uso pesado de reflection**

* Normal em frameworks (Ex: DI tipo Dagger, serializadores JSON).
* Anômalo quando:

  * Chamadas constantes a `Class.forName`, `Method.invoke` em métodos triviais.
  * Strings de nomes de classe/método ofuscadas ou criptografadas.
* Motivo: esconder chamadas sensíveis (ex.: APIs privadas, loaders de código dinâmico).
* Defesa: instrumentar chamadas reflection → logar parâmetros → ver se está chamando APIs suspeitas.

### 2. **DexClassLoader em runtime**

* Permite carregar `.dex` ou `.apk` do storage ou rede em tempo real.
* Uso legítimo: plugin architecture (raro).
* Uso típico em malware: baixar payload criptografado e injetar como código.
* Indicador claro de evasão estática: app original parece inofensivo, mas vira outra coisa depois.

### 3. **Blobs AES que viram classes**

* Arquivos binários aparentemente “dados” no assets ou res/raw.
* Em runtime, o app chama `Cipher.getInstance("AES/…")`, decripta e carrega via DexClassLoader.
* Assim o payload não aparece em análises estáticas.
* Se o blob decripta para um `.dex`, `.so` ou `.jar`, sinal fortíssimo de comportamento malicioso.

### 4. **Verificação obsessiva de build props e sensores**

* Ler `Build.MODEL`, `Build.FINGERPRINT`, IMEI, lista de apps, sensores disponíveis, etc.
* Quando feito uma vez → ok (analytics, debug).
* Quando repetido, em loops ou antes de ativar payload → é sandbox detection.
* Exemplo: só ativa se achar WhatsApp instalado, ou se `Build.FINGERPRINT` não contiver `generic`.

### 5. **Timers longos sem propósito funcional**

* App que agenda `Handler.postDelayed` ou `AlarmManager.setExact` para “nada”.
* Malware usa isso para atrasar execução (ex.: só ligar payload depois de 24h de instalação).
* Serve para enganar sandboxes que rodam só alguns minutos.

### 6. **Recepção de muitos broadcasts do sistema**

* Apps legítimos registram 1 ou 2 (`BOOT_COMPLETED`, `CONNECTIVITY_CHANGE`).
* Malware registra dezenas: screen on/off, SIM swap, package install, SMS recebido, etc.
* Objetivo: renascer sempre que o SO dispara algo, mantendo persistência mesmo que o usuário feche o app.

---

...
