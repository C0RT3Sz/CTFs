📄 **RELATÓRIO TÉCNICO – CTF MR. ROBOT (TryHackMe)**

👤 **Autor:** c0rt3s
🎯 **Plataforma:** TryHackMe
🖥️ **Máquina:** Mr. Robot
📌 **Tipo:** CTF – Web / Linux / Privilege Escalation
⚙️ **Objetivo:** Obter as 3 flags da máquina
📊 **Nível de dificuldade:** Médio
🧠 **Base de estudo:** Conteúdo e metodologia do curso da Desec Security

---

🎯 Objetivo do Lab

O objetivo deste laboratório foi comprometer a máquina *Mr. Robot* da plataforma TryHackMe e obter as três flags solicitadas, aplicando técnicas de reconhecimento, enumeração web, força bruta, exploração de WordPress, obtenção de shell reversa e escalonamento de privilégios em ambiente Linux.

A máquina foi escolhida como forma de validar, na prática, os conhecimentos adquiridos durante os estudos na Desec Security.

---

 🔍 Reconhecimento Inicial – Enumeração de Portas

Como em todo CTF, o primeiro passo foi identificar os serviços expostos pelo alvo.

Foi realizado um scan com Nmap:

```
nmap -v -sS -sV 10.64.166.14
```

O scan identificou as seguintes portas abertas:

* 22 (SSH)
* 80 (HTTP)
* 443 (HTTPS)

A presença da porta 80 indicava um serviço web ativo, que passou a ser o foco inicial da exploração.

---

🌐 Enumeração Web – Gobuster

Ao acessar o serviço HTTP pelo navegador, foi encontrado um site temático do Mr. Robot. Com isso, foi iniciada a enumeração de diretórios e arquivos utilizando o Gobuster:

```
gobuster dir -u http://10.64.166.14 -w /home/c0rt3s/SecLists/Discovery/Web-Content/common.txt -a "Mozila 5.1"
```

Durante a enumeração, dois pontos chamaram atenção:

* Arquivo `robots.txt`
* Diretório `wp-admin`, indicando o uso de WordPress

---

 🤖 Análise do robots.txt

Ao acessar o arquivo `robots.txt`, foram encontradas duas informações críticas:

* A **primeira flag** (key-1-of-3.txt)
* Um arquivo chamado `fsocity.dic`

O arquivo `fsocity.dic` foi baixado com o comando:

```
wget http://10.64.166.14/fsocity.dic
```

Após análise, ficou claro que se tratava de uma **wordlist**, indicando que seria utilizada em um ataque de força bruta contra o WordPress.

---

 🔐 Ataque de Força Bruta – WordPress

Inicialmente foi realizado brute force para identificar usuários válidos utilizando o próprio arquivo `fsocity.dic`:

```
wpscan --url http://10.64.166.14/wp-login.php fsocity.dic --username fsocity.dic
```

Após bastante tempo de execução, foi identificado o usuário válido:

* **Usuário:** elliot

Em seguida, foi realizado brute force da senha utilizando o Hydra:

```
hydra -V -t 8 -l elliot -P fsocity_clean.dic 10.64.166.14 http-post-form \
"/xmlrpc.php:<?xml version='1.0'?><methodCall><methodName>wp.getUsersBlogs</methodName><params><param><value><string>elliot</string></value></param><param><value><string>^PASS^</string></value></param></params></methodCall>:faultCode"
```

Após um período longo de execução, a senha foi descoberta:

* **Senha:** ER28-0652

Com isso, foi possível acessar o painel administrativo do WordPress.

**OBS: Depois de ter terminado a maquina eu descobri que eu conseguiria usar o Burp Suite para descobrir o usuario mais rapido**

---

🧨 Exploração – Reverse Shell via WordPress

Após acessar o painel administrativo, foi utilizado o **Editor de Aparência** do WordPress para modificar arquivos do tema.

Foi inserido um código de **reverse shell em PHP** (Pentestmonkey) no arquivo `archive.php`.

No atacante, foi iniciado um listener:

```
nc -lvnp 1337
```

Ao acessar o arquivo `archive.php` pelo navegador, a conexão reversa foi estabelecida com sucesso, concedendo acesso ao sistema.

---

 🐚 Acesso Inicial ao Sistema

Com a shell reversa ativa, foi identificado que o acesso inicial ocorreu como o usuário atual do serviço.

Foi então acessado o diretório:

```
/home/robot
```

Dentro desse diretório foram encontrados:

* `key-2-of-3.txt` (sem permissão de leitura)
* `password.raw-md5`

O arquivo `password.raw-md5` continha um hash MD5 referente à senha do usuário `robot`.

Após a quebra do hash, foi possível realizar:

```
su robot
```

Com isso, foi obtido acesso ao usuário `robot` e à **segunda flag**.

---

🔎 Enumeração de Privilégios – Binários SUID

Para obter acesso root, foi realizada enumeração de binários com permissão SUID:

```
find / -perm -4000 -type f 2>/dev/null
```

Durante a enumeração, foi identificado um binário crítico:

```
/usr/local/bin/nmap
```

Esse binário possuía o bit SUID ativo.

---

🚀 Escalonamento de Privilégio – Nmap SUID

Foi executado o Nmap em modo interativo:

```
/usr/local/bin/nmap --interactive
```

Dentro do modo interativo, foi executado:

```
!sh
```

Devido ao bit SUID, o shell foi iniciado com privilégios de **root**.

---

🔓 Acesso Root e Flag Final

Com acesso root, foi possível acessar o diretório:

```
/root
```

Dentro dele, foi localizada a **terceira e última flag** (`key-3-of-3.txt`), finalizando com sucesso a máquina Mr. Robot.

---

🧠 Conclusão

Este laboratório demonstrou claramente a importância de uma metodologia sólida de pentest. A máquina Mr. Robot apresentou diversas falhas clássicas:

* Vazamento de informações via `robots.txt`
* WordPress vulnerável a brute force
* Execução de código via editor de temas
* Binário SUID mal configurado permitindo escalonamento de privilégio

A conclusão bem-sucedida deste CTF reforça que os estudos realizados na Desec Security estão trazendo resultados práticos e consistentes, consolidando tanto a parte teórica quanto a prática em ambientes reais de exploração.
