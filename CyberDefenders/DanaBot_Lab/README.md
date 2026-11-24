# 🛡️ RELATÓRIO DE INVESTIGAÇÃO FORENSE (DFIR) – LAB DANABOT (CyberDefenders)

👤 **Autor:** c0rt3s  
🎯 **Plataforma:** CyberDefenders  
📦 **Artefato Analisado:** `205-DanaBot.pcap`  
📌 **Tipo:** Blue Team – Network Forensics / Malware Traffic Analysis / Threat Hunting  
⚙️ **Objetivo:** Investigar a cadeia de infecção do malware DanaBot exclusivamente via CLI (`tshark`) e extrair indicadores de comprometimento (IoCs)  
📊 **Nível de dificuldade:** Fácil  
🧠 **Base de estudo:** Prática de análise de tráfego em linha de comando (`tshark` + utilitários Linux)

---

# 🎯 Objetivo da Investigação

O objetivo deste laboratório foi conduzir uma investigação de perícia em rede (*Network Forensics*) sobre uma captura de tráfego (`205-DanaBot.pcap`) contendo a cadeia de infecção inicial do trojan bancário/loader **DanaBot**.

Toda a investigação foi executada exclusivamente via **linha de comando (CLI)** no Linux, utilizando o `tshark` e ferramentas nativas (`grep`, `sha256sum`, `md5sum` e `file`), sem apoio de interface gráfica (Wireshark), com foco na análise de protocolos, reconstrução de fluxos HTTP/TCP e extração de evidências.

---

# 🔗 Resumo da Cadeia de Comprometimento (Kill Chain Observada)

A análise do tráfego de rede e dos artefatos extraídos permitiu reconstruir o seguinte fluxo de infecção:

```text
Acesso inicial via HTTP (GET /login.php)
        ↓
Download do stager JavaScript ofuscado (allegato_708.js)
        ↓
Execução via Windows Script Host (wscript.exe)
        ↓
Requisição secundária ao servidor de payload (GET /resources.dll)
        ↓
Gravação e carregamento da DLL maliciosa (rundll32.exe)
        ↓
Auto-exclusão do stager inicial (anti-forense básico)
```

Esse fluxo caracteriza uma **cadeia de entrega em múltiplos estágios (*multi-stage delivery*)**, na qual um script inicial atua como downloader para buscar o payload subsequente.

---

# 🔍 Triagem Inicial – Identificação de Fluxos e Acesso Inicial

O primeiro passo foi filtrar os protocolos presentes na captura e inspecionar as requisições HTTP realizadas pelo host comprometido (`10.2.14.101`) utilizando o `tshark`:

```bash
tshark -r 205-DanaBot.pcap -Y "http.request" -T fields -e tcp.stream -e ip.src -e ip.dst -e http.host -e http.request.uri
```

Fluxos suspeitos identificados:

- **Stream 0:** `10.2.14.101` → `62.173.142.148` (`portfolio.serveirc.com`) — `GET /login.php`
- **Stream HTTP secundário:** `10.2.14.101` → `188.114.97.3` (`soundata.top`) — `GET /resources.dll`

A comunicação inicial com o IP **`62.173.142.148`**, associado ao domínio `portfolio.serveirc.com`, chamou atenção por entregar um payload sob uma URI que simulava uma página de autenticação (`/login.php`).

---

# 📂 Inspeção de Cabeçalhos e Identificação do Stager

Para inspecionar a conversa completa da camada de aplicação no **Stream 0**, foi utilizado o recurso de reconstrução de fluxo do `tshark`:

```bash
tshark -r 205-DanaBot.pcap -q -z follow,http,ascii,0
```

Evidências encontradas nos cabeçalhos HTTP de resposta (`HTTP/1.1 200 OK`):

```http
Content-disposition: attachment;filename=allegato_708.js
Transfer-Encoding: chunked
```

Descobertas técnicas:

- O servidor forçou o download do arquivo com o nome **`allegato_708.js`** através do cabeçalho `Content-Disposition`.
- O corpo da resposta utilizou transferência `chunked` e continha código JavaScript fortemente ofuscado.

---

# 🧪 Extração de Artefatos e Validação de Integridade

Para extrair os arquivos transferidos via HTTP, removendo os cabeçalhos de resposta e remontando os segmentos TCP e chunks automaticamente, foi utilizado o exportador de objetos do `tshark`:

```bash
tshark -r 205-DanaBot.pcap -q --export-objects http,objetos_http
ls -la objetos_http/
```

Artefatos maliciosos extraídos:

1. **`login.php`** – arquivo correspondente ao stager JavaScript `allegato_708.js` — `5.558 bytes`
2. **`resources.dll`** – payload binário secundário — `11.922.432 bytes`

Cálculo dos hashes criptográficos para catalogação de IoCs e consulta em bases de *Threat Intelligence*:

```bash
sha256sum objetos_http/login.php
md5sum objetos_http/resources.dll
file objetos_http/resources.dll
```

Resultados obtidos:

- **Stager (`allegato_708.js` / `login.php`):**
  - **SHA-256:** `847b4ad90b1daba2d9117a8e05776f3f902dda593fb1252289538acf476c4268`

- **Payload (`resources.dll`):**
  - **Tipo:** `PE32 executable (DLL) (GUI) Intel 80386, for MS Windows`
  - **MD5:** `e758e07113016aca55d9eda2b0ffeebe`
  - **SHA-256:** `2597322a49a6252445ca4c8d713320b238113b3b8fd8a2d6fc1088a5934cee0e`

---

# 🧬 Análise Estática do Stager (`allegato_708.js`)

Embora o arquivo `.pcap` registre apenas o tráfego de rede, a análise estática do script extraído permitiu identificar mecanismos de execução e comportamento previstos no artefato, sem necessidade de executá-lo em sandbox:

```bash
grep -oE '(WScript|ActiveXObject)[^;]+' objetos_http/login.php
```

Chamadas críticas identificadas no código ofuscado:

- Instanciação de **`WScript`** e **`ActiveXObject`**: indica execução via **Windows Script Host (`wscript.exe`)**.
- Uso de `Scripting.FileSystemObject` (`GetSpecialFolder(2)`) para direcionar o download do segundo estágio (`resources.dll`) ao diretório `%TEMP%`.
- Uso de `ADODB.Stream` para gravação binária da DLL em disco e `WScript.Shell` para acionar sua execução.
- Chamada final `DeleteFile(WScript.ScriptFullName)` para **auto-exclusão do próprio script JavaScript**, dificultando sua coleta posterior no disco da vítima.

---

# 📊 Tabela de Indicadores de Comprometimento (IoCs)

| Tipo de IoC | Indicador | Contexto / Estágio |
|---|---|---|
| **IPv4** | `62.173.142.148` | Servidor de entrega inicial (Stager JS) |
| **Domínio** | `portfolio.serveirc.com` | Host HTTP do acesso inicial (`GET /login.php`) |
| **Arquivo (Estágio 1)** | `allegato_708.js` | Script downloader ofuscado |
| **SHA-256 (Estágio 1)** | `847b4ad90b1daba2d9117a8e05776f3f902dda593fb1252289538acf476c4268` | Hash do stager |
| **Processo Executor** | `wscript.exe` | Interpretador Windows Script Host |
| **IPv4** | `188.114.97.3` | Servidor de distribuição do payload secundário |
| **Domínio** | `soundata.top` | Host HTTP do payload (`GET /resources.dll`) |
| **Arquivo (Estágio 2)** | `resources.dll` | Payload principal (DLL PE32) |
| **MD5 (Estágio 2)** | `e758e07113016aca55d9eda2b0ffeebe` | Hash MD5 da DLL |
| **SHA-256 (Estágio 2)** | `2597322a49a6252445ca4c8d713320b238113b3b8fd8a2d6fc1088a5934cee0e` | Hash SHA-256 da DLL |

---

# 💥 Impacto de Segurança

Em um ambiente corporativo real, a execução bem-sucedida dessa cadeia de infecção poderia resultar em:

- Execução de código no contexto do usuário por meio de mecanismos como `wscript.exe` e `rundll32.exe`.
- Instalação do trojan **DanaBot**, associado a atividades de roubo de credenciais, informações e sessões.
- Dificuldade de análise posterior no endpoint devido à auto-exclusão do stager JavaScript.

---

# 🛡️ Controles Defensivos e Recomendações (Detecção & Hardening)

- **Hardening de Endpoint:** restringir ou controlar a execução de scripts via Windows Script Host, utilizando políticas como AppLocker ou WDAC, quando aplicável.
- **Regras de Detecção (SIEM / EDR):** monitorar processos filhos suspeitos originados de `wscript.exe`, especialmente quando associados a `rundll32.exe` ou `powershell.exe`, além de criação de arquivos `.dll` em diretórios como `%TEMP%` e `%APPDATA%`.
- **Filtragem de Proxy / DNS / IDS:** avaliar o bloqueio dos IoCs identificados (`62.173.142.148`, `188.114.97.3`, `portfolio.serveirc.com` e `soundata.top`), restringir downloads suspeitos e monitorar domínios dinâmicos (DDNS) e domínios potencialmente abusados.

---

# 🧠 Conclusão

Esta investigação demonstrou na prática como a análise de tráfego em linha de comando com `tshark` permite reconstruir etapas de uma cadeia de infecção, desde as requisições HTTP iniciais até o comportamento do script responsável pela entrega do payload subsequente.

Principais aprendizados consolidados:

- Filtragem de requisições HTTP e identificação de fluxos suspeitos utilizando `-Y` e `-T fields`.
- Reconstrução de fluxos HTTP com `-z follow,http,ascii`.
- Extração automatizada de objetos HTTP e validação de integridade por meio de hashes criptográficos.
- Análise estática de scripts obfuscados para identificar mecanismos de execução, download, gravação e exclusão de arquivos.
- Correlação entre evidências de rede e artefatos de aplicação para reconstruir o contexto de uma atividade maliciosa.
- Estruturação de IoCs e identificação de possíveis controles defensivos e mecanismos de detecção.

---
