# 📄 RELATÓRIO TÉCNICO – CTF RETRO (TryHackMe)

👤 **Autor:** c0rt3s
🎯 **Plataforma:** TryHackMe
🖥️ **Máquina:** Retro
📌 **Tipo:** CTF – Windows / WordPress / Privilege Escalation
⚙️ **Objetivo:** Obter acesso inicial, escalar privilégios e capturar as flags
📊 **Nível de dificuldade:** Difícil
🧠 **Base de estudo:** Conteúdo e metodologia do curso da Desec Security e metodologia DUNGEON

---

# 🎯 Objetivo do Lab

O objetivo deste laboratório foi comprometer a máquina **Retro** explorando falhas de segurança em um servidor **Windows Server 2016** rodando **WordPress** e **RDP**.

Durante o processo foram utilizadas técnicas de:

* Reconhecimento de serviços
* Enumeração web
* Enumeração WordPress
* Descoberta de credenciais
* Reutilização de credenciais (Credential Reuse)
* Acesso remoto via RDP
* Enumeração local
* Escalonamento de privilégio no Windows

A exploração culminou na obtenção de acesso **SYSTEM**, permitindo a captura das flags do sistema.

---

# ⚔️ Resumo da Cadeia de Ataque

A exploração seguiu o seguinte fluxo:

```
Nmap Scan
      ↓
Enumeração Web
      ↓
Descoberta do diretório /retro
      ↓
Enumeração WordPress (WPScan)
      ↓
Usuário wade identificado
      ↓
Senha encontrada em comentário
      ↓
Login no WordPress
      ↓
Reverse shell limitada (iusr)
      ↓
Teste da credencial no RDP
      ↓
Acesso RDP como wade
      ↓
Enumeração local
      ↓
Exploit CVE-2017-0213
      ↓
Shell SYSTEM
      ↓
Captura das flags
```

Essa cadeia representa um ataque baseado em **credenciais expostas e reutilização de senha para acesso remoto**, seguido de escalonamento local de privilégios.

---

# 🔍 Reconhecimento Inicial – Nmap

O primeiro passo foi identificar os serviços disponíveis na máquina alvo.

```
nmap -sS -Pn -n -p- --reason retro.thm
nmap -A -Pn -n -sC -sV -p- retro.thm
```

Portas identificadas:

```
80/tcp   HTTP
3389/tcp RDP
```

Serviços identificados:

```
Microsoft IIS 10.0
Microsoft Terminal Services (RDP)
```

Isso indicava um **servidor Windows com acesso remoto habilitado**.

---

# 🌐 Enumeração Web

Foi realizada enumeração de diretórios utilizando Gobuster.

```
gobuster dir -u http://retro.thm -w directory-list-2.3-medium.txt
```

Diretórios encontrados:

```
/retro
/Retro
```

O diretório **/retro** redirecionava para um site **WordPress**.

---

# 📝 Enumeração WordPress – WPScan

Foi utilizada a ferramenta WPScan para enumeração do WordPress.

```
wpscan --url http://retro.thm/retro/ --enumerate
```

Informações identificadas:

* WordPress 5.2.1
* PHP 7.1.29
* Tema: 90s-retro
* XML-RPC habilitado
* Página de login disponível
* Usuário identificado: **wade**

---

# 🔑 Descoberta de Credenciais

Durante a análise dos posts do WordPress, foi encontrado um comentário no post **"Ready Player One"** contendo uma senha.

Senha encontrada:

```
parzival
```

Credencial válida:

```
Usuário: wade
Senha: parzival
```

Essa credencial funcionava tanto no **WordPress** quanto no **RDP**, caracterizando **reutilização de credenciais**.

---

# 🐚 Obtenção de Shell via WordPress

Após acessar o painel administrativo do WordPress, foi inserida uma webshell no arquivo:

```
archive.php
```

A webshell abriu uma reverse shell PowerShell.

Shell obtida como:

```
nt authority\iusr
```

Esse usuário possui privilégios muito limitados, o que restringia a exploração local através da reverse shell.

Devido às limitações da shell, a credencial encontrada foi testada no serviço **RDP**.

---

# 🖥️ Acesso via RDP

A credencial foi utilizada para acessar a máquina via RDP.

```
Usuário: wade
Senha: parzival
```

O login foi bem-sucedido, permitindo acesso interativo completo ao sistema Windows, o que facilitou a enumeração local e o processo de escalonamento de privilégio.

---

# 🔎 Enumeração Local

Durante a enumeração local foram identificados:

* Diretório `C:\badr` com permissão de escrita
* Executável `badr.exe` rodando como SYSTEM
* Possibilidade de DLL Hijacking
* Sistema vulnerável a exploits de elevação de privilégio

Foram testadas várias técnicas de escalonamento:

* DLL Hijacking
* PrintSpoofer
* JuicyPotato
* CVE-2019-1388
* CVE-2017-0213

---

# 🚀 Escalonamento de Privilégio

O método que funcionou foi:

```
CVE-2017-0213 — Windows COM Elevation of Privilege
```

O exploit foi executado como o usuário **wade** e resultou em uma shell com privilégios:

```
NT AUTHORITY\SYSTEM
```

Isso concedeu controle total da máquina.

---

# 🚩 Captura das Flags

Flags encontradas:

```
C:\Users\Wade\Desktop\user.txt.txt
C:\Users\Administrator\Desktop\root.txt.txt
```
---

# 💥 Impacto de Segurança

Se esse cenário estivesse presente em um ambiente corporativo real, um atacante poderia:

* obter acesso ao WordPress
* descobrir credenciais expostas
* reutilizar credenciais para acesso RDP
* obter acesso remoto ao servidor Windows
* escalar privilégios para SYSTEM
* comprometer totalmente o servidor
* acessar banco de dados e arquivos internos
* usar o servidor como pivot para rede interna

---

# 🛡️ Mitigações Recomendadas

Algumas medidas que poderiam prevenir esse tipo de ataque incluem:

* Remover credenciais expostas em comentários públicos
* Não reutilizar senhas entre sistemas
* Implementar autenticação multifator no RDP
* Atualizar WordPress para versões mais recentes
* Aplicar patches de segurança do Windows
* Corrigir permissões de diretórios com escrita para usuários comuns
* Restringir acesso RDP via firewall
* Monitorar tentativas de login remoto

---

# 🧠 Conclusão

Este laboratório demonstrou diversas falhas comuns em servidores Windows com aplicações web:

* credenciais expostas em aplicação web
* reutilização de credenciais entre serviços
* acesso remoto via RDP com senha reutilizada
* permissões incorretas em diretórios
* sistema vulnerável a exploits de elevação de privilégio

A exploração mostrou que **o ponto crítico do comprometimento não foi a webshell, mas a reutilização de credenciais que permitiu acesso via RDP**, facilitando a escalada de privilégios até **SYSTEM**.

Este cenário representa uma falha muito comum em ambientes corporativos reais, onde uma única credencial exposta pode levar ao comprometimento completo de um servidor Windows.

