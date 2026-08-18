<h1 align="center">Do Local à Nuvem: Projeto BIA com Docker, WSL 2 e AWS</h1>

<p align="center">
  <strong>Desafio 1 — Formação AWS 5.0</strong><br>
  Ambiente de desenvolvimento containerizado no Windows com WSL 2, aplicação Node.js + React com PostgreSQL persistido,<br>
  e a mesma stack publicada em uma instância EC2 gerenciada por SSM.
</p>

<p align="center">
  <img alt="Docker" src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white">
  <img alt="Docker Compose" src="https://img.shields.io/badge/Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white">
  <img alt="PostgreSQL" src="https://img.shields.io/badge/PostgreSQL%2017-4169E1?style=for-the-badge&logo=postgresql&logoColor=white">
  <img alt="Node.js" src="https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white">
  <img alt="React" src="https://img.shields.io/badge/React-61DAFB?style=for-the-badge&logo=react&logoColor=black">
</p>

<p align="center">
  <img alt="AWS" src="https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonwebservices&logoColor=white">
  <img alt="Amazon EC2" src="https://img.shields.io/badge/EC2-FF9900?style=for-the-badge&logo=amazonec2&logoColor=white">
  <img alt="Amazon RDS" src="https://img.shields.io/badge/RDS-527FFF?style=for-the-badge&logo=amazonrds&logoColor=white">
  <img alt="Ubuntu" src="https://img.shields.io/badge/Ubuntu%2024.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white">
  <img alt="WSL 2" src="https://img.shields.io/badge/WSL%202-0078D4?style=for-the-badge&logo=windows&logoColor=white">
</p>

---

## Índice

- [Objetivo](#-objetivo)
- [Resultado](#-resultado)
- [Arquitetura](#-arquitetura)
  - [Visão geral: local × nuvem](#visão-geral-local--nuvem)
  - [Arquitetura local (WSL 2 / Docker)](#arquitetura-local-wsl-2--docker)
  - [Arquitetura na AWS](#arquitetura-na-aws)
- [Ambiente da máquina](#-ambiente-da-máquina)
- [Configuração local passo a passo](#-configuração-local-passo-a-passo)
- [Etapa extra: subindo para a AWS](#-etapa-extra-subindo-para-a-aws)
- [Checklist de entrega](#-checklist-de-entrega)
- [O que aprendi](#-o-que-aprendi)
- [Créditos e referências](#-créditos-e-referências)

---

## 🎯 Objetivo

Montar do zero um ambiente de desenvolvimento completo, instalar as dependências e colocar o projeto
**BIA** (Node.js + React) para rodar localmente via Docker — aplicação web e banco relacional
PostgreSQL — **persistindo os dados** e com o banco acessível externamente pelo **DBeaver**.

Como **etapa extra**, provisionar uma instância **EC2 Linux na AWS** e publicar a mesma aplicação
na nuvem, usando **RDS** como banco gerenciado.

O que foi entregue:

- ✅ Ambiente Linux sobre **WSL 2 + Ubuntu 24.04 LTS**, dimensionado para um notebook de 8 GB de RAM
- ✅ **Git + autenticação SSH no GitHub** configurados dentro do Ubuntu
- ✅ **VS Code** conectado ao WSL (Remote — WSL), trabalhando no filesystem Linux
- ✅ **Docker Desktop** integrado ao WSL 2, sem um segundo daemon Docker dentro do Ubuntu
- ✅ Projeto BIA rodando com **Docker Compose**: aplicação (`3001`), PostgreSQL (`5433`) e Redis/Valkey (`6379`)
- ✅ **Persistência de dados** validada — a tarefa criada sobrevive ao restart dos containers
- ✅ **DBeaver** conectado ao PostgreSQL do container pela porta mapeada
- ✅ **EC2 `bia-dev`** provisionada via **CloudShell**, acessada por **SSM Session Manager** (sem SSH, sem key pair)
- ✅ **Security Group** e **IAM Role** dedicados, seguindo o princípio do menor privilégio
- ✅ **AWS Budget** configurado antes de qualquer recurso ser criado

---

## 📸 Resultado

A mesma aplicação, nos dois ambientes:

| Local — `localhost:3001` | AWS — EC2 `bia-dev` na porta `3001` |
| :---: | :---: |
| <img src="imagens/Localhost_3001.jpg" alt="Aplicação BIA rodando em localhost:3001 com uma tarefa persistida no banco"> | <img src="imagens/AWS_3001.png" alt="Aplicação BIA rodando na instância EC2 pela porta 3001"> |
| Registro gravado no PostgreSQL do container e mantido após reiniciar o ambiente | Mesma imagem da aplicação servida a partir da instância EC2 na região `us-east-1` |

---

## 🏗️ Arquitetura

### Visão geral: local × nuvem

![Comparativo entre o ambiente local em WSL/Docker e o ambiente AWS](imagens/Arquitetura_Local_e_em_Nuvem.png)

| Elemento | Ambiente local (Docker) | Ambiente AWS |
| --- | --- | --- |
| Porta da aplicação | `8080` interna, exposta em `3001` | `3001` exposta pelo Security Group |
| Banco de dados | PostgreSQL em container (`5432` → `5433`) | **RDS PostgreSQL** gerenciado |
| Rede | Rede bridge do Docker dentro do WSL 2 | VPC com Security Group `bia-dev` |
| Método de acesso | Browser + DBeaver via `localhost` | Browser + **SSM Session Manager** / Console |
| Provisionamento | `docker compose up -d` | AWS CloudShell + AWS CLI |

O ponto central do desafio é perceber que **a aplicação não muda** — o que muda é a camada de
infraestrutura embaixo dela. O contrato `container → porta → banco` é o mesmo nos dois lados.

### Arquitetura local (WSL 2 / Docker)

![Arquitetura local Docker: usuário acessa a porta 3001 e o DBeaver a porta 5433, ambos mapeados para containers na rede Docker do WSL](imagens/Arquitetura_localhost.jpg)

| Componente | Porta interna | Porta no host (WSL) | Papel |
| --- | :---: | :---: | --- |
| **SERVER** (`bia-server`) | `8080` | `3001` | Container da aplicação — front-end React + API Node.js |
| **DATABASE** (`postgres:17.1`) | `5432` | `5433` | PostgreSQL com os dados da aplicação |
| **REDIS** (`valkey:8.1-alpine`) | `6379` | `6379` | Cache da aplicação |
| **DBeaver** | — | conecta em `5433` | Client de banco rodando no Windows |

Dois detalhes que valem atenção:

- **A aplicação não usa a porta `5433`.** O container `server` fala com o `database` pela rede
  interna do Docker, na porta nativa `5432`, resolvendo o nome do serviço. A `5433` existe apenas
  para o DBeaver acessar o banco de fora — e foi escolhida justamente para não colidir com uma
  eventual instalação de PostgreSQL no host.
- **O Redis/Valkey é o terceiro container** do Compose. Ele não aparece no diagrama original, mas
  sobe junto e é visível no `docker compose ps` mais abaixo.

### Arquitetura na AWS

![Arquitetura AWS: CloudShell provisiona a EC2 bia-dev na VPC, com Security Group, IAM Role e acesso via SSM](imagens/Arquitetura_AWS.jpg)

| Componente | Localização | Função na arquitetura |
| --- | --- | --- |
| **Browser / User** | Externo | Acesso ao console da AWS e à aplicação publicada |
| **GitHub** | Externo | Repositório com o código/scripts, clonado a partir do CloudShell |
| **CloudShell** | AWS (ambiente base) | Shell Linux gerenciado (1 GB storage / 2 GB RAM) usado para validar recursos, criar a role e lançar a EC2 |
| **SSM (Systems Manager)** | AWS (gerenciado) | Acesso administrativo à EC2 via SSM Agent — **sem abrir a porta 22 e sem key pair** |
| **EC2 `bia-dev`** | VPC | Instância `t3.micro` (`us-east-1a`) que executa o container da aplicação |
| **Security Group `bia-dev`** | VPC | Regra de entrada liberando apenas a porta `3001` |
| **IAM Role `role-acesso-ssm`** | IAM | Autorização que permite à instância conversar com os serviços da AWS |
| **RDS PostgreSQL** | VPC | Banco relacional gerenciado consumido pela aplicação na EC2 |
| **ECR / ECS** | AWS | Destinos naturais da evolução: publicar a imagem no ECR e orquestrar no ECS |

> A escolha por **SSM Session Manager no lugar de SSH** é o que mais mudou minha forma de pensar
> acesso a instâncias: não existe porta 22 aberta, não existe chave privada para vazar, e todo acesso
> passa por IAM — auditável e revogável.

---

## 🧰 Ambiente da máquina

Este laboratório foi executado em um notebook modesto — e isso influenciou cada decisão de arquitetura.

| Hardware | Especificação |
| --- | --- |
| Notebook | Dell Inspiron 14 3000 |
| Processador | Intel Core i3-4005U @ 1.70 GHz — 2 núcleos / 4 threads |
| Memória | 8 GB DDR3 |
| Sistema | Windows 11 Pro 24H2 |

| Stack | Versão |
| --- | --- |
| WSL | 2.7.11.0 |
| Ubuntu | 24.04.4 LTS |
| Git | 2.43.0 |
| Docker Desktop | 4.86.0 |
| Docker Engine | 29.7.2 |
| Docker Compose | v5.3.1 |

![Docker Desktop mostrando as imagens locais: bia-server, valkey, nginx, hello-world e postgres 17.1](imagens/Docker.desktop.jpg)

### Por que WSL 2 e não uma VM tradicional

A instrução original do curso sugere montar uma VM para o treinamento. Com 8 GB de RAM e um i3 de
2 núcleos, subir um sistema operacional completo no VirtualBox significaria manter dois SOs
disputando a mesma memória:

```text
VirtualBox                          WSL 2
Windows                             Windows
  └── VM Ubuntu (SO completo)         └── Ubuntu (kernel compartilhado)
        └── Docker Engine                   └── Docker CLI
                                                  └── Docker Desktop (engine único)
```

Optei pelo WSL 2 e **limitei explicitamente seus recursos** via `.wslconfig` (3 GB de RAM, 2 vCPUs,
1 GB de swap), deixando ~5 GB para Windows, navegador, VS Code e Docker Desktop. O resultado é um
ambiente Linux real, com Docker nativo, que não derruba a máquina hospedeira.

O mesmo raciocínio se aplica ao Docker: **não** instalei `docker.io` dentro do Ubuntu. O Docker
Desktop já fornece o engine, e a integração com o WSL apenas expõe a CLI dentro da distribuição —
um daemon, não dois.

---

## ⚙️ Configuração local passo a passo

> A ordem importa: cada camada é validada antes da próxima ser instalada. Se algo falhar, você sabe
> exatamente onde.

### 1. Validar o Windows e o estado atual

No **PowerShell** (usuário normal):

```powershell
winver          # confirmar Windows 10/11 com suporte a WSL 2
wsl --status    # versão padrão do WSL e distro default
wsl --version   # versão do WSL, kernel e WSLg
docker version  # verificar se o Docker Desktop já está instalado
```

Se o `docker version` retornar erro de conexão com a API, normalmente significa apenas que o
**Docker Desktop não está em execução** — não que a instalação esteja quebrada.

### 2. Instalar o Ubuntu 24.04 no WSL 2

```powershell
wsl --list --online     # distribuições disponíveis
wsl --list --verbose    # o que já está instalado
wsl --install Ubuntu-24.04
```

Na primeira execução o Ubuntu pede usuário e senha UNIX.

> ⚠️ Ao digitar a senha no Linux **nada aparece na tela**. Isso é o comportamento esperado.

Validação dentro do Ubuntu:

```bash
uname -a
cat /etc/os-release
free -h
nproc
```

Confirme no PowerShell que a distro está em **VERSION 2**:

```powershell
wsl --list --verbose
```

```text
  NAME              STATE           VERSION
* Ubuntu-24.04      Running         2
  docker-desktop    Stopped         2
```

### 3. Limitar os recursos do WSL (`.wslconfig`)

Este é o passo que torna o laboratório viável em 8 GB de RAM.

```powershell
notepad "$env:USERPROFILE\.wslconfig"
```

```ini
[wsl2]
memory=3GB
processors=2
swap=1GB
localhostForwarding=true

[experimental]
autoMemoryReclaim=gradual
```

| Parâmetro | Efeito |
| --- | --- |
| `memory=3GB` | Teto de RAM do WSL — impede que o Ubuntu sufoque o Windows |
| `processors=2` | 2 das 4 threads para o WSL, mantendo o host responsivo |
| `swap=1GB` | Proteção contra estouro de memória (mais lento que RAM, mas evita OOM) |
| `localhostForwarding=true` | **Essencial aqui**: faz `localhost:3001` do Windows chegar no container dentro do WSL |
| `autoMemoryReclaim=gradual` | Devolve ao Windows a memória que o WSL deixou de usar |

O arquivo só vale após reiniciar o WSL por completo:

```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

Verifique dentro do Ubuntu:

```bash
free -h    # esperado: ~3.0Gi de Mem e 1.0Gi de Swap
nproc      # esperado: 2
```

### 4. Instalar e configurar o Git no Ubuntu

O Git fica **dentro do Ubuntu**, não no Windows.

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install git -y
git --version
```

```bash
git config --global user.name "Seu Nome"
git config --global user.email "seu.email@exemplo.com"
git config --global --list
```

### 5. Autenticar no GitHub via SSH

```bash
ls -la ~/.ssh                                     # verificar se já existe uma chave
ssh-keygen -t ed25519 -C "seu.email@exemplo.com"  # Enter para aceitar o caminho padrão
```

Recomendo definir uma **passphrase** para a chave. Em seguida, carregue-a no agente:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

Copie **apenas a chave pública**:

```bash
cat ~/.ssh/id_ed25519.pub
```

No GitHub: **Settings → SSH and GPG keys → New SSH key**, tipo *Authentication Key*. Teste:

```bash
ssh -T git@github.com
```

> ⚠️ `id_ed25519` é a **chave privada** e nunca deve ser compartilhada, commitada ou colada em
> documentação. Apenas o arquivo `.pub` vai para o GitHub.

### 6. Conectar o VS Code ao WSL

Pré-requisitos do VS Code Server dentro do Ubuntu:

```bash
sudo apt install wget ca-certificates -y
dpkg -l ca-certificates | grep '^ii'
wget -q --spider https://update.code.visualstudio.com && echo $?   # esperado: 0
```

No VS Code (Windows): `Ctrl + Shift + X` → instalar a extensão **WSL (Microsoft)** →
`Ctrl + Shift + P` → **WSL: Connect to WSL**. O canto inferior esquerdo passa a exibir
`WSL: Ubuntu-24.04`.

Estrutura de trabalho dentro do Linux:

```bash
mkdir -p ~/projetos ~/repos ~/laboratorios-aws
```

> ⚠️ Mantenha os projetos em `/home/usuario/...` e **não** em `/mnt/c/Users/...`. Trabalhar no
> filesystem montado do Windows degrada bastante a performance de Git, Docker e ferramentas Linux.

### 7. Integrar o Docker Desktop ao WSL 2

> ⚠️ **Não** execute `sudo apt install docker.io` nem `docker-ce` dentro do Ubuntu. Isso cria um
> segundo daemon Docker, concorrente com o do Docker Desktop e caro demais para 8 GB de RAM.

No Docker Desktop: **Settings → Resources → WSL Integration** → marcar
`Enable integration with my default WSL distro` e a distro **`Ubuntu-24.04`** → **Apply & Restart**.

Reinicie a distribuição para carregar a integração:

```powershell
wsl --terminate Ubuntu-24.04
wsl -d Ubuntu-24.04
```

Valide dentro do Ubuntu — o importante é aparecerem **Client e Server**:

```bash
docker version
docker info
docker context ls
docker run hello-world
```

### 8. Smoke test do Docker Compose

Antes de subir a aplicação real, um teste mínimo confirma o mapeamento de portas ponta a ponta:

```bash
mkdir -p ~/laboratorios-aws/docker-hello && cd ~/laboratorios-aws/docker-hello
```

```yaml
# compose.yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
```

```bash
docker compose up -d
docker compose ps
curl http://localhost:8080   # deve retornar o HTML padrão do Nginx
docker compose down
```

### 9. Subir o projeto BIA

```bash
cd ~/laboratorios-aws
git clone https://github.com/henrylle/bia
cd bia
docker compose up -d
docker compose ps
```

![Terminal exibindo docker compose up -d e docker compose ps com os containers bia, database e redis em execução](imagens/Localhost_terminal.jpg)

```text
NAME       IMAGE                      SERVICE    STATUS     PORTS
bia        bia-server                 server     Up         0.0.0.0:3001->8080/tcp
database   postgres:17.1              database   Up         0.0.0.0:5433->5432/tcp
redis      valkey/valkey:8.1-alpine   redis      Up         0.0.0.0:6379->6379/tcp
```

Acesse **<http://localhost:3001>** no navegador do Windows — graças ao `localhostForwarding=true`,
a porta publicada dentro do WSL responde direto no host.

![Aplicação BIA aberta em localhost:3001 com uma tarefa cadastrada](imagens/Localhost_3001.jpg)

### 10. Conectar o DBeaver ao banco

Com o container `database` no ar, crie uma conexão PostgreSQL no DBeaver:

| Campo | Valor |
| --- | --- |
| Host | `localhost` |
| Porta | `5433` |
| Database / Usuário / Senha | conforme as variáveis definidas no `compose.yaml` do projeto |

Use **Test Connection** antes de salvar. Se falhar, confirme que o container está `Up` e que a porta
`5433` aparece no `docker compose ps`.

### 11. Encerrar o ambiente

```bash
docker compose down      # remove containers e rede — os volumes (e os dados) permanecem
docker compose down -v   # remove também os volumes — apaga os dados do banco
```

A diferença entre os dois comandos é exatamente o que o desafio pede para comprovar: rode
`docker compose down`, suba tudo de novo e verifique que a tarefa cadastrada continua lá.

---

## ☁️ Etapa extra: subindo para a AWS

Com o ambiente local validado, a mesma stack foi publicada na nuvem.

**1. Conta e controle de custo (antes de tudo)**

- Conta criada com **Free Tier**
- **Billing and Cost Management → Budgets → Monthly cost budget** configurado antes de provisionar
  qualquer recurso
- Região de trabalho: **`us-east-1` (N. Virginia)**

**2. Preparação no console e no CloudShell**

- **Security Group `bia-dev`** — regra de entrada liberando a porta `3001`
- **IAM Role `role-acesso-ssm`** — autoriza a instância a conversar com o Systems Manager; sem ela,
  a EC2 simplesmente não aparece no Session Manager
- **AWS CloudShell** (1 GB storage / 2 GB RAM, já com `aws cli`, `git`, `python` e `node`) usado para
  validar recursos, criar a role e lançar a instância

**3. Instância EC2**

![Console da AWS mostrando a instância bia-dev t3.micro em estado Running com 3/3 status checks](imagens/console-aws-amazon-ec2.png)

| Atributo | Valor |
| --- | --- |
| Nome | `bia-dev` |
| Tipo | `t3.micro` |
| Zona de disponibilidade | `us-east-1a` |
| Status checks | 3/3 passed |
| Acesso | **SSM Session Manager** (sem porta 22, sem key pair) |
| Security Group | `bia-dev` — inbound `3001` |
| IAM Role | `role-acesso-ssm` |

**4. Aplicação e banco**

- Docker instalado na instância e repositório clonado
- Container da aplicação publicado no mapeamento **`3001:8080`**, o mesmo do ambiente local
- **RDS PostgreSQL** como banco gerenciado, no lugar do container usado localmente
- Acesso final: `http://<IP-PUBLICO-DA-EC2>:3001`

![Aplicação BIA respondendo a partir da EC2 na porta 3001](imagens/AWS_3001.png)

### ⚠️ Encerrando os recursos

Laboratório concluído é laboratório destruído. Ao final:

1. **Terminate** da instância EC2
2. **Delete** da instância RDS (atenção ao snapshot final, que também é cobrado)
3. Remover Security Group e Role se não forem reaproveitados
4. Conferir o **Budget** e o Cost Explorer no dia seguinte

---

## ✅ Checklist de entrega

| # | Item | Status |
| :-: | --- | :-: |
| 1 | WSL 2 instalado e validado | ✅ |
| 2 | Ubuntu 24.04 LTS como distro de trabalho | ✅ |
| 3 | `.wslconfig` limitando RAM, CPU e swap | ✅ |
| 4 | Git instalado e configurado no Ubuntu | ✅ |
| 5 | Autenticação SSH no GitHub funcionando | ✅ |
| 6 | VS Code conectado ao WSL | ✅ |
| 7 | Docker Desktop integrado ao WSL 2 | ✅ |
| 8 | Docker Compose validado | ✅ |
| 9 | Projeto BIA rodando em `localhost:3001` | ✅ |
| 10 | Persistência de dados comprovada | ✅ |
| 11 | DBeaver conectado ao PostgreSQL em `5433` | ✅ |
| 12 | AWS Budget configurado | ✅ |
| 13 | Security Group `bia-dev` (porta 3001) | ✅ |
| 14 | IAM Role `role-acesso-ssm` | ✅ |
| 15 | EC2 `bia-dev` lançada via CloudShell | ✅ |
| 16 | Acesso à instância via SSM Session Manager | ✅ |
| 17 | Aplicação publicada na EC2 com RDS | ✅ |

---

## 🧠 O que aprendi

- **Mapeamento de portas não é detalhe.** Entender que `3001:8080` significa "host:container" e que
  a aplicação conversa com o banco pela porta *interna* (`5432`), não pela publicada (`5433`), é o
  que separa "copiei o comando" de "entendi a rede".
- **Persistência mora no volume, não no container.** `docker compose down` e
  `docker compose down -v` parecem o mesmo comando até a primeira vez que os dados somem.
- **Dimensionar recursos é uma decisão de arquitetura.** Escolher WSL 2 no lugar de uma VM completa
  e limitar o ambiente a 3 GB foi o que tornou o laboratório possível em um notebook de 8 GB.
- **Acesso sem SSH é mais seguro e mais simples.** Com SSM Agent + IAM Role não há porta 22 aberta
  nem chave privada circulando, e todo acesso fica auditável.
- **Permissão vem antes de conectividade.** Uma EC2 sem a role correta não aparece no Session
  Manager — não é problema de rede, é de IAM.
- **Custo é responsabilidade técnica.** Budget configurado antes do primeiro recurso e destruição do
  ambiente ao final fazem parte do trabalho, não são um extra.

---

## 📚 Créditos e referências

- **Formação AWS 5.0** — Prof. Henrylle Maia
- Projeto BIA — <https://github.com/henrylle/bia>
- [WSL — documentação oficial da Microsoft](https://learn.microsoft.com/windows/wsl/)
- [Docker Desktop — WSL 2 integration](https://docs.docker.com/desktop/features/wsl/)
- [AWS Systems Manager — Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)

---

<p align="center">
  <sub>Os nomes, e-mails e caminhos de usuário deste documento são <strong>placeholders</strong>. Substitua-os pelos seus ao reproduzir o passo a passo.</sub>
</p>
