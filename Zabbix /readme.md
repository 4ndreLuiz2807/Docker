
# Manual completo — Zabbix 8.0 LTS com Docker e Docker Compose

> **Instalação do zero, passo a passo, em Ubuntu Server ou Debian**  
> **Arquitetura:** Zabbix Server 8.0 LTS + Zabbix Web (Nginx) + PostgreSQL 17  
> **Porta da interface:** `8085/TCP` | **Porta do servidor Zabbix:** `10051/TCP`  
> **Atualizado em:** 08/11/2026 | **Idioma:** Português (Brasil)

Este documento acompanha o pacote `Zabbix_Docker_8_LTS.zip` e explica **o que cada ferramenta faz**, **por que cada comando é executado**, como realizar a instalação, testar a comunicação, preservar o banco, atualizar e resolver problemas comuns. O guia foi elaborado para instalação em um servidor Linux, sem substituir ou reconfigurar outros serviços já existentes (por exemplo, GLPI).

> **Importante:** a versão 8.0 LTS foi lançada em 05/11/2026. As imagens `alpine-8.0-latest` recebem novas atualizações **dentro da linha 8.0**. Antes de usar em produção, valide a implantação e defina backup, HTTPS e controle de acesso. Este pacote foi conferido estaticamente; os containers **não foram executados** no ambiente de criação desta documentação.

---

## Índice

1. [O que será instalado](#1-o-que-será-instalado)
2. [Ferramentas e conceitos essenciais](#2-ferramentas-e-conceitos-essenciais)
3. [Requisitos e planejamento](#3-requisitos-e-planejamento)
4. [Conhecer os arquivos do pacote](#4-conhecer-os-arquivos-do-pacote)
5. [Identificar o sistema operacional](#5-identificar-o-sistema-operacional)
6. [Instalar Docker Engine e Compose no Ubuntu](#6-instalar-docker-engine-e-compose-no-ubuntu)
7. [Instalar Docker Engine e Compose no Debian](#7-instalar-docker-engine-e-compose-no-debian)
8. [Verificar se o Docker funciona](#8-verificar-se-o-docker-funciona)
9. [Transferir e extrair o pacote](#9-transferir-e-extrair-o-pacote)
10. [Entender a configuração do Compose](#10-entender-a-configuração-do-compose)
11. [Entender o arquivo .env](#11-entender-o-arquivo-env)
12. [Iniciar os containers do Zabbix](#12-iniciar-os-containers-do-zabbix)
13. [Validar a instalação](#13-validar-a-instalação)
14. [Acessar a interface web](#14-acessar-a-interface-web)
15. [Conhecer portas, redes e segurança](#15-conhecer-portas-redes-e-segurança)
16. [Adicionar o primeiro equipamento ao Zabbix](#16-adicionar-o-primeiro-equipamento-ao-zabbix)
17. [Comandos de administração](#17-comandos-de-administração)
18. [Backup e restauração do PostgreSQL](#18-backup-e-restauração-do-postgresql)
19. [Atualizações e manutenção](#19-atualizações-e-manutenção)
20. [Solução de problemas](#20-solução-de-problemas)
21. [Checklist e próximos passos](#21-checklist-e-próximos-passos)
22. [Referências oficiais](#22-referências-oficiais)

---

## 1. O que será instalado

Serão criados **3 containers**:

| Serviço do Compose | Imagem utilizada | Função | Exposição no host |
|---|---|---|---|
| `postgres` | `postgres:17-alpine` | Armazena configurações, eventos e dados históricos | **Não publicada** |
| `zabbix-server` | `zabbix/zabbix-server-pgsql:alpine-8.0-latest` | Executa a coleta, processa dados e dispara alertas | `10051/TCP` |
| `zabbix-web` | `zabbix/zabbix-web-nginx-pgsql:alpine-8.0-latest` | Interface gráfica para administração e dashboards | `8085/TCP` → `8080/TCP` |

### Como os componentes se comunicam

```text
                         REDE LOCAL
                             |
                Navegador (PC ou notebook)
                             |
                   http://IP:8085
                             |
                    +------------------+
                    | zabbix-web       |
                    | Nginx + frontend |
                    +------------------+
                         |         |
                         |         +------------------------+
                         v                                  v
                 +------------------+              +------------------+
Agentes ativos ->| zabbix-server    |<------------>| postgres         |
  TCP 10051      | Motor do Zabbix  |              | Banco PostgreSQL |
                 +------------------+              +------------------+
                         ^                                  |
                         |                           Volume Docker
                 ICMP / SNMP / Agent                  postgres_data

         Comunicação interna: rede privada criada pelo Compose.
         O banco de dados não publica a porta 5432 no host.
```

**Por que três containers em vez de um?** Separar os serviços facilita reiniciar um componente, investigar erros e atualizar sem misturar as responsabilidades do servidor, do frontend e do banco.

**O que não vem neste pacote?** Zabbix Agent 2 no host Linux, agentes Windows, Zabbix Proxy, Java Gateway, traps SNMP, HTTPS e proxy reverso. Esses recursos podem ser configurados depois.

## 2. Ferramentas e conceitos essenciais

### 2.1 Docker

**Docker** é uma plataforma que executa aplicações isoladas em *containers*. Um container contém os arquivos e dependências necessários à execução da aplicação, mas compartilha o kernel do Linux. É diferente de uma máquina virtual completa, que normalmente executa seu próprio sistema operacional convidado.

- **Docker Engine:** serviço que cria e controla os containers no servidor Linux.
- **Docker CLI:** comando `docker`, usado para administrar imagens, containers e volumes.
- **Imagem:** modelo pronto e versionado, como `postgres:17-alpine`.
- **Container:** instância em execução de uma imagem.
- **Docker Hub:** repositório de onde o Docker baixa as imagens públicas.

**Exemplo:** `sudo docker run --rm hello-world` baixa uma imagem de demonstração, executa-a e remove o container após finalizar.

### 2.2 Docker Compose

**Docker Compose** é o recurso que permite declarar vários serviços num único arquivo YAML (`compose.yaml`). Em vez de executar manualmente muitos comandos `docker run`, usamos:

```bash
sudo docker compose up -d
```

Esse comando lê a configuração, cria a rede e o volume, baixa as imagens que faltarem e inicia os containers. **Docker é o mecanismo de execução; Compose é a ferramenta de orquestração local de vários containers.**

Use a sintaxe atual **`docker compose`** (com espaço), oferecida pelo plugin Compose v2. Não é necessário instalar o antigo `docker-compose` com hífen.

> **Não confundir com Kubernetes:** para esta instalação simples, Docker Compose é suficiente. Kubernetes é outra plataforma, voltada a uma orquestração mais ampla entre nós e clusters; não é obrigatório para rodar o Zabbix.

### 2.3 PostgreSQL

Banco de dados relacional utilizado para armazenar hosts, configurações, itens, eventos e métricas. Neste projeto, os dados são guardados no volume `postgres_data`.

### 2.4 Zabbix Server

Processa verificações, recebe dados de agentes/proxies, avalia triggers e envia eventos/alertas. O servidor central usa a porta `10051/TCP` para determinadas comunicações com agentes ativos e proxies.

### 2.5 Zabbix Web + Nginx

É o painel acessado pelo navegador. O Nginx recebe conexões HTTP dentro do container na porta `8080`; o Docker publica essa interface no **host** como porta `8085`.

### 2.6 Volume Docker

É um armazenamento persistente, gerenciado pelo Docker e independente do ciclo de vida do container. Mesmo ao recriar o container PostgreSQL com `docker compose up -d`, o conteúdo do volume continua disponível.

**Atenção:** persistência **não é backup**. Excluir o volume ou perder o disco do servidor pode causar perda definitiva de dados.

### 2.7 Rede Docker

O Compose cria uma rede privada normalmente chamada `zabbix_default`. Os containers se comunicam pelo **nome do serviço** (`postgres`, `zabbix-server`, `zabbix-web`), sem precisar fixar IPs internos.

### 2.8 Ferramentas auxiliares

| Ferramenta/comando | Finalidade |
|---|---|
| `sudo` | Executa comandos com privilégios administrativos |
| `apt` | Instala e atualiza pacotes no Ubuntu/Debian |
| `curl` | Baixa a chave pública do repositório Docker e testa serviços HTTP |
| `ca-certificates` | Certificados necessários à validação de conexões HTTPS |
| `unzip` | Extrai os arquivos do pacote ZIP |
| `openssl` | Gera uma senha aleatória forte para o banco |
| `nano` | Editor de texto opcional para alterar a configuração |
| `systemctl` | Gerencia o serviço Docker no systemd |
| `ss` | Inspeciona portas em escuta no host |
| `hostname -I` | Mostra endereços IP do servidor |

## 3. Requisitos e planejamento

- Ubuntu Server ou Debian 64 bits, com acesso administrativo (`sudo` ou root).
- Docker Engine e Docker Compose v2 (a documentação do Zabbix 8.0 indica Compose **2.24.0 ou superior**).
- Internet no momento de instalar pacotes e baixar imagens do Docker Hub.
- Recomenda-se, como **ponto de partida para laboratório**, pelo menos 2 vCPU, 4 GiB RAM e 20 GiB de armazenamento livre. O consumo real depende da quantidade de hosts, itens coletados, histórico e retenção; não é um dimensionamento de produção.
- IP fixo ou reserva DHCP para o servidor, preferencialmente.
- Portas `8085/TCP` (painel) e `10051/TCP` (tráfego adequado do Zabbix) disponíveis.

**Antes de começar, verifique se o servidor já tem outros containers.** Este guia não remove os existentes. Se Docker já está instalado e funcionando, **pule a instalação do Engine** e siga para a etapa 8.

**Exemplo de endereçamento** (substitua pelo IP verdadeiro da sua máquina):

```text
Servidor Linux:  192.168.1.50
Interface:       http://192.168.1.50:8085
Servidor Zabbix: 192.168.1.50:10051
```

## 4. Conhecer os arquivos do pacote

```text
Zabbix_Docker_8_LTS.zip
├── README.md       <- Este manual
├── compose.yaml    <- Declara os 3 containers, rede, volume e portas
├── .env.example    <- Modelo das variáveis de configuração
└── instalar.sh     <- Instalador que cria .env e executa o Compose
```

**Funções:**

- **`compose.yaml`:** configuração principal, em YAML. Não contém a senha do banco em texto fixo.
- **`.env`:** criado durante a instalação; contém a senha do banco e a porta web. **É privado e não deve ir para GitHub.**
- **`.env.example`:** exemplo sem credenciais reais. Pode ser compartilhado.
- **`instalar.sh`:** valida pré-requisitos, gera senha segura e inicia a implantação. Se `.env` já existir, o script o preserva.
- **`README.md`:** roteiro de consulta e recuperação do ambiente.

## 5. Identificar o sistema operacional

No terminal do servidor, execute:

```bash
cat /etc/os-release
uname -m
```

**O que fazem:** `cat /etc/os-release` mostra o nome e a versão do Linux. `uname -m` mostra a arquitetura, como `x86_64` ou `aarch64`.

Depois, escolha **somente um** dos procedimentos abaixo:

- Ubuntu → etapa **6**;
- Debian → etapa **7**.

> **Instalação existente:** se `sudo docker --version` e `sudo docker compose version` já funcionam, não troque a instalação atual sem necessidade. Pule para a etapa 8.

## 6. Instalar Docker Engine e Compose no Ubuntu

Procedimento baseado no repositório **oficial** da Docker. Indicado para instalações novas compatíveis (Ubuntu 22.04, 24.04, 26.04 LTS e versões indicadas na documentação atual).

### Passo 6.1 — Atualizar o índice de pacotes

```bash
sudo apt update
sudo apt install -y ca-certificates curl unzip openssl nano
```

**Explicação:** `apt update` consulta as listas dos repositórios. O segundo comando instala certificados, ferramenta de download, descompactador, geração de senha e editor de arquivos.

### Passo 6.2 — Preparar a pasta das chaves e obter a chave GPG

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

**Explicação:** cria um diretório para chaves, baixa a chave oficial do repositório e permite que o APT leia essa chave. A validação por assinatura ajuda a autenticar os pacotes baixados.

### Passo 6.3 — Adicionar o repositório oficial do Ubuntu

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

**Explicação:** registra o servidor de pacotes oficial da Docker usando a versão e arquitetura detectadas. A última linha atualiza a lista, incluindo o novo repositório.

### Passo 6.4 — Instalar os componentes

```bash
sudo apt install -y \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

**Significado:** `docker-ce` = Engine, `docker-ce-cli` = terminal, `containerd.io` = runtime, `docker-buildx-plugin` = construção avançada de imagens, `docker-compose-plugin` = `docker compose`.

> **Se houver conflito de pacotes:** algumas instalações anteriores usam `docker.io`, `docker-compose` ou `containerd` do sistema. Leia o guia oficial antes de substituí-los, principalmente em servidores que já hospedam containers. **Não execute uma remoção de pacotes às cegas.**

### Passo 6.5 — Habilitar o serviço

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

**Resultado esperado:** serviço `active (running)`. O parâmetro `enable --now` habilita o Docker no boot e tenta iniciá-lo imediatamente.

**Agora vá para a etapa 8.**

## 7. Instalar Docker Engine e Compose no Debian

Procedimento para o repositório **oficial** da Docker em Debian 12 (Bookworm) ou Debian 13 (Trixie).

### Passo 7.1 — Dependências

```bash
sudo apt update
sudo apt install -y ca-certificates curl unzip openssl nano
```

### Passo 7.2 — Chave do repositório

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### Passo 7.3 — Repositório Docker para Debian

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

**Explicação:** é similar ao Ubuntu, mas utiliza o repositório `linux/debian` e o codinome Debian (`bookworm` ou `trixie`).

### Passo 7.4 — Instalar e iniciar

```bash
sudo apt install -y \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

**Resultado esperado:** Docker com estado `active (running)`.

## 8. Verificar se o Docker funciona

Execute os quatro testes:

```bash
sudo docker --version
sudo docker compose version
sudo docker info
sudo docker run --rm hello-world
```

| Comando | O que comprova |
|---|---|
| `docker --version` | CLI do Docker instalada |
| `docker compose version` | Plugin Compose instalado |
| `docker info` | Cliente consegue se comunicar com o Docker Engine |
| `docker run --rm hello-world` | Imagem pode ser baixada e executada |

O `hello-world` deve mostrar uma mensagem contendo **“Hello from Docker!”**. É um teste, não um container permanente. Se funcionar, continue.

> **Por que usamos `sudo`?** O socket do Docker exige permissões elevadas. Adicionar usuários ao grupo `docker` concede privilégios equivalentes a root em muitas situações; por isso, este manual utiliza `sudo` em vez de recomendar essa alteração.

## 9. Transferir e extrair o pacote

### Passo 9.1 — Enviar o ZIP ao Linux

Transfira o arquivo `Zabbix_Docker_8_LTS.zip` para `/tmp` do servidor Linux por SCP, SFTP ou pelo gerenciador de arquivos. Por exemplo, em um **PowerShell do Windows**, com SSH disponível no Linux:

```powershell
scp "$env:USERPROFILE\Downloads\Zabbix_Docker_8_LTS.zip" usuario@IP_DO_SERVIDOR:/tmp/
```

**Explicação:** `scp` copia o ZIP da pasta Downloads do Windows para o diretório temporário do Linux. Substitua `usuario` e `IP_DO_SERVIDOR`. Se o ZIP já estiver no Linux, não precisa executar `scp`.

### Passo 9.2 — Conferir o arquivo e preparar a pasta

```bash
ls -lh /tmp/Zabbix_Docker_8_LTS.zip
sudo mkdir -p /opt/zabbix-docker
```

**Explicação:** `ls -lh` confirma que o arquivo existe. `mkdir -p` cria a pasta definitiva; `/opt` é um local adequado para aplicativos administrados no servidor.

### Passo 9.3 — Extrair e listar o conteúdo

```bash
sudo unzip /tmp/Zabbix_Docker_8_LTS.zip -d /opt/zabbix-docker
cd /opt/zabbix-docker
ls -la
```

**Explicação:** `unzip` extrai os quatro arquivos; `cd` entra na pasta do projeto; `ls -la` exibe inclusive `.env.example`, que é oculto por começar com ponto.

**Não use `unzip -o` sobre uma instalação já personalizada sem revisar os arquivos**: ele pode sobrescrever o Compose e o instalador existentes.

## 10. Entender a configuração do Compose

Abra o arquivo principal:

```bash
cd /opt/zabbix-docker
cat compose.yaml
```

### Principais instruções do `compose.yaml`

| Instrução | Significado no pacote |
|---|---|
| `name: zabbix` | Nome do projeto Compose; prefixo para recursos gerados |
| `services:` | Lista das três aplicações/containeres |
| `image:` | Imagem Docker exata usada para cada serviço |
| `restart: unless-stopped` | Reinicia automaticamente após falhas/reboot, salvo se parado manualmente |
| `environment:` | Variáveis de ambiente entregues ao aplicativo |
| `ports:` | Liga uma porta do host a uma porta do container |
| `depends_on:` | Define a ordem/condições de inicialização |
| `healthcheck:` | Verifica se o PostgreSQL está aceitando conexões |
| `volumes:` | Persiste dados, mesmo ao recriar o container |
| `logging:` | Limita o tamanho e a quantidade de arquivos de log |
| `init: true` | Adiciona processo de inicialização leve para melhor gerenciamento de processos |

### Serviço `postgres`

- Inicia PostgreSQL 17.
- Cria o banco `zabbix` com usuário `zabbix` na **primeira** inicialização de um volume vazio.
- Utiliza a senha guardada no `.env`.
- Salva os arquivos de dados no volume `postgres_data`.
- Executa `pg_isready` periodicamente para verificar a prontidão.
- Não utiliza `ports:`: portanto, não expõe o PostgreSQL na rede do host.

### Serviço `zabbix-server`

- Usa a imagem oficial para PostgreSQL.
- `DB_SERVER_HOST: postgres` localiza o banco pelo **nome de serviço** na rede interna.
- Aguarda o banco ficar saudável (`service_healthy`) antes de iniciar.
- Publica `10051:10051`, permitindo conexões apropriadas de agentes ativos e proxies.

### Serviço `zabbix-web`

- Executa a interface Zabbix no Nginx.
- `ZBX_SERVER_HOST: zabbix-server` aponta para o backend.
- `PHP_TZ: America/Sao_Paulo` configura o fuso horário do PHP.
- `"${ZABBIX_WEB_PORT:-8085}:8080"` significa: publicar a porta definida em `.env` ou usar `8085` se ela não estiver definida; a porta interna é `8080`.

### Sobre a rede e o volume

Sem definir uma rede manual no arquivo, o Compose cria automaticamente a rede `zabbix_default`. O volume declarado `postgres_data` será normalmente criado com o nome `zabbix_postgres_data`.

Para listar o volume depois da instalação:

```bash
sudo docker volume ls
sudo docker network ls
```

## 11. Entender o arquivo .env

**`.env`** define valores substituídos pelo Docker Compose. Neste projeto, contém:

```dotenv
POSTGRES_PASSWORD=SENHA_FORTE_GERADA_PELO_INSTALADOR
ZABBIX_WEB_PORT=8085
```

### Opção recomendada — deixar o instalador gerar a senha

Se **não existir** `.env`, `instalar.sh` gera uma senha aleatória com:

```bash
openssl rand -hex 24
```

O resultado são 24 bytes aleatórios representados por 48 caracteres hexadecimais. O script salva o arquivo com permissões restritas (`chmod 600`). **Não publique essa senha nem compartilhe o arquivo `.env`.**

### Opção manual — escolher uma senha própria antes da instalação

```bash
cd /opt/zabbix-docker
sudo cp .env.example .env
sudo nano .env
sudo chmod 600 .env
```

No editor, substitua `SUBSTITUA_POR_UMA_SENHA_FORTE` por sua senha, salve com `Ctrl+O` + `Enter` e saia com `Ctrl+X`. Evite caracteres especiais que possam interferir na interpretação de variáveis sem o devido cuidado; uma senha hexadecimal longa é uma opção simples.

### Alterar a porta web

Para usar, por exemplo, `8090` em vez de `8085`, edite a variável:

```dotenv
ZABBIX_WEB_PORT=8090
```

Depois recrie apenas o serviço necessário ou aplique a configuração:

```bash
sudo docker compose up -d
```

**Importante sobre a senha do banco:** mudar `POSTGRES_PASSWORD` no `.env` **não altera automaticamente** a senha dentro de um PostgreSQL já inicializado. Não mude essa variável numa instalação em uso sem executar o procedimento correto de alteração da senha do banco.

## 12. Iniciar os containers do Zabbix

### Passo 12.1 — Entrar na pasta

```bash
cd /opt/zabbix-docker
```

Todos os próximos comandos `docker compose` devem rodar dentro dessa pasta (ou utilizar explicitamente `-f` e `--project-directory`).

### Passo 12.2 — Executar o instalador

```bash
sudo bash instalar.sh
```

**O que acontece, na ordem:**

1. O script muda para a pasta onde foi colocado.
2. Verifica se `docker` existe.
3. Verifica se o plugin `docker compose` funciona.
4. Se não houver `.env`, cria uma senha PostgreSQL segura com OpenSSL.
5. Ajusta permissão do `.env` para `600`.
6. Executa `docker compose config --quiet` para validar a configuração sem imprimir a senha.
7. Executa `docker compose up -d` para baixar imagens, criar volume/rede e iniciar os containers em segundo plano.
8. Exibe `docker compose ps` e as informações de acesso.

**Alternativa manual, após criar `.env`:**

```bash
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose up -d
```

- `config --quiet`: testa sintaxe e interpolação sem exibir configurações sensíveis;
- `pull`: baixa as imagens especificadas;
- `up -d`: cria e inicia os serviços sem prender o terminal.

> **Primeira execução:** o PostgreSQL e o Zabbix Server precisam criar o esquema de dados. A interface pode levar alguns minutos para ficar pronta, conforme o hardware e a conexão.

## 13. Validar a instalação

### Passo 13.1 — Ver containers

```bash
cd /opt/zabbix-docker
sudo docker compose ps
```

Você deverá identificar os serviços `postgres`, `zabbix-server` e `zabbix-web` em estado de execução. O PostgreSQL deve ficar **healthy** após os testes de prontidão.

### Passo 13.2 — Acompanhar logs

```bash
sudo docker compose logs --tail=100 postgres
sudo docker compose logs --tail=100 zabbix-server
sudo docker compose logs --tail=100 zabbix-web
```

Para ver novos eventos em tempo real:

```bash
sudo docker compose logs -f zabbix-server
```

Use `Ctrl+C` para sair do acompanhamento; isso **não para o container**.

### Passo 13.3 — Conferir IP do host

```bash
hostname -I
```

**Explicação:** mostra os IPs atribuídos às interfaces de rede. Se aparecerem vários endereços, use o IP da interface acessível pela sua rede local.

### Passo 13.4 — Testar a interface no próprio servidor

```bash
curl -I http://127.0.0.1:8085
```

**Resultado esperado:** uma resposta HTTP, por exemplo `200` ou um redirecionamento `302`. A ausência temporária de resposta pode indicar que a inicialização ainda está em andamento.

Se você alterou a porta, substitua `8085` pelo valor de `ZABBIX_WEB_PORT`.

### Passo 13.5 — Conferir comunicação com o PostgreSQL

```bash
sudo docker compose exec -T postgres pg_isready -U zabbix -d zabbix
```

**Resultado esperado:** `accepting connections`. O comando usa o binário dentro do container PostgreSQL; não exige instalar o cliente PostgreSQL no host.

## 14. Acessar a interface web

No navegador de outro equipamento na mesma rede, entre em:

```text
http://IP_DO_SERVIDOR:8085
```

No próprio servidor (se houver navegador):

```text
http://localhost:8085
```

**Credenciais iniciais:**

| Campo | Valor |
|---|---|
| Usuário | `Admin` |
| Senha | `zabbix` |

Após o primeiro acesso:

1. **Altere imediatamente** a senha do usuário `Admin` no perfil do Zabbix.
2. Ajuste idioma, fuso e preferências conforme necessário.
3. Verifique no painel se o **Zabbix server está em execução**.
4. Cadastre seus primeiros hosts para iniciar as coletas.

> A porta `8085` serve HTTP, **sem HTTPS**. Para acesso externo, use VPN ou proxy reverso TLS/HTTPS com autenticação e controles de rede. Evite expor a porta diretamente na internet.

## 15. Conhecer portas, redes e segurança

| Porta | Sentido / função | Publicada pelo pacote? |
|---|---|---|
| `8085/TCP` | Navegador → Zabbix Web (internamente `8080`) | Sim |
| `10051/TCP` | Agentes ativos/proxies → Zabbix Server | Sim |
| `10050/TCP` | Zabbix Server → agente passivo em host monitorado | **Não**; depende dos agentes externos |
| `5432/TCP` | Zabbix Server/Web → PostgreSQL | Não, somente rede Docker |
| `161/UDP` | Consultas SNMP a equipamentos monitorados | Não é porta publicada pelo Zabbix; é o destino nos equipamentos |

### Validar portas no host

```bash
sudo docker compose ps
sudo ss -lntp | grep -E ':(8085|10051)\b' || true
```

**Observação:** dependendo da configuração do Docker, `ss` pode não mostrar todas as portas publicadas, mesmo quando estão funcionais; `docker compose ps` e testes de conexão são mais confiáveis para confirmar o mapeamento.

### Acesso apenas pela rede local

O Compose do pacote usa a publicação padrão do Docker, que pode escutar em todas as interfaces. Se quiser restringir a interface **ao próprio servidor**, troque no `compose.yaml`:

```yaml
ports:
  - "127.0.0.1:${ZABBIX_WEB_PORT:-8085}:8080"
```

Com isso, não será possível abrir o Zabbix diretamente de outro computador; é necessário um proxy reverso local ou túnel seguro. Para acesso remoto controlado, prefira VPN e regras no firewall de borda/rede.

**Alerta importante:** portas publicadas pelo Docker podem não respeitar regras simples do `ufw` como seria esperado para outros serviços. Revise a documentação oficial de **Docker e firewall**, as regras `DOCKER-USER` e o firewall externo antes de publicar serviços em redes não confiáveis.

### Cuidados adicionais

- Use senhas únicas e fortes; não envie `.env` em chamados ou repositórios.
- Proteja o acesso SSH ao servidor e mantenha o sistema atualizado.
- Faça backup do banco **fora do servidor** e teste a restauração.
- Evite executar `docker compose down -v`.
- Mantenha o horário do sistema e NTP corretos.
- Para produção, considere autenticação centralizada e HTTPS em proxy reverso.

## 16. Adicionar o primeiro equipamento ao Zabbix

Instalar a central **não adiciona automaticamente todos os computadores**. Você precisa cadastrar os hosts e escolher a forma de monitoramento.

### Opção A — Monitorar um equipamento por ICMP (ping)

1. Entre no painel Zabbix com uma conta administrativa.
2. Vá até **Coleta de dados → Hosts → Criar host** (o nome do menu pode mudar com o idioma/versão).
3. Defina um nome descritivo e selecione um grupo de hosts.
4. Vincule o template de ICMP, quando disponível, como `ICMP Ping`.
5. Configure a interface/endereço necessário ao template e informe o IP ou DNS do equipamento.
6. Salve e confira os dados em **Monitoramento → Dados recentes** ou disponibilidade.

**O que mede:** se o host responde a ping. Isso **não** informa CPU, memória ou disco.

### Opção B — Monitorar Windows ou Linux com Zabbix Agent 2

1. Instale um **Zabbix Agent 2** compatível no equipamento monitorado, seguindo o instalador oficial desse sistema.
2. Configure o hostname do agente exatamente como o nome cadastrado no Zabbix quando usar verificações ativas.
3. Para verificações **ativas**, configure `ServerActive=IP_DO_ZABBIX:10051`; o agente inicia conexão com a central.
4. Para verificações **passivas**, permita a comunicação do server até a porta `10050/TCP` do agente e configure `Server=IP_DO_ZABBIX`.
5. No painel, associe o template adequado, por exemplo `Windows by Zabbix agent` ou `Linux by Zabbix agent`.
6. Verifique **Dados recentes** após a primeira coleta.

**Atenção:** o IP visto pelo Agent em monitoramentos passivos pode depender da rede/roteamento do Docker. Em ambientes corporativos, valide endereçamento, NAT, TLS e firewall antes de ampliar a implantação.

### Opção C — Monitorar switches, roteadores e impressoras por SNMP

Habilite SNMP nos equipamentos, dê preferência a **SNMPv3** e crie os hosts no Zabbix com interfaces SNMP e templates apropriados. Evite comunidades SNMPv2c padrão em redes produtivas.

## 17. Comandos de administração

Execute em `/opt/zabbix-docker`, salvo quando indicado:

```bash
cd /opt/zabbix-docker
```

| Objetivo | Comando | Efeito |
|---|---|---|
| Estado geral | `sudo docker compose ps` | Mostra os três serviços |
| Ver todos os containers do host | `sudo docker ps -a` | Exibe também outros projetos |
| Logs do server | `sudo docker compose logs --tail=150 zabbix-server` | Ajuda a investigar falhas |
| Acompanhar logs | `sudo docker compose logs -f` | Mostra logs em tempo real |
| Reiniciar o Zabbix | `sudo docker compose restart` | Reinicia os serviços sem apagar dados |
| Parar projeto | `sudo docker compose stop` | Para containers, preservando-os |
| Iniciar projeto parado | `sudo docker compose start` | Reinicia containers já criados |
| Remover containers/rede do projeto | `sudo docker compose down` | **Mantém volumes** por padrão |
| Criar/iniciar novamente | `sudo docker compose up -d` | Recria se necessário |
| Validar YAML | `sudo docker compose config --quiet` | Valida sem revelar segredos no terminal |
| Ver imagens | `sudo docker images` | Lista imagens baixadas |
| Ver volumes | `sudo docker volume ls` | Lista volumes persistentes |
| Ver uso de disco | `sudo docker system df` | Resumo do armazenamento Docker |

### Comandos que exigem extremo cuidado

```bash
# NÃO executar para manutenção cotidiana:
sudo docker compose down -v
sudo docker volume prune
sudo docker system prune --volumes
```

Eles podem remover volumes persistentes do Zabbix ou **de outros projetos**, causando perda de dados. Não execute sem um backup verificável e entendimento das consequências.

## 18. Backup e restauração do PostgreSQL

### 18.1 Por que o backup é necessário?

O volume garante persistência quando o container é recriado, mas não protege de exclusão de volume, erro humano, corrupção, ransomware ou falha de disco. O backup lógico com `pg_dump` é a forma inicial mais simples de proteger o banco.

### 18.2 Criar uma pasta de backups

```bash
sudo mkdir -p /var/backups/zabbix
sudo chmod 700 /var/backups/zabbix
cd /opt/zabbix-docker
```

**Explicação:** os arquivos são guardados fora do diretório de instalação e com acesso restrito.

### 18.3 Fazer backup com `pg_dump`

```bash
sudo bash -c 'docker compose exec -T postgres \
  pg_dump -U zabbix -d zabbix -Fc \
  > /var/backups/zabbix/zabbix-$(date +%F-%H%M).dump'
```

**O que acontece:**

- `docker compose exec -T postgres`: executa um comando no container sem alocar terminal interativo;
- `pg_dump -U zabbix -d zabbix`: exporta o banco do Zabbix;
- `-Fc`: formato customizado, compatível com `pg_restore`;
- `>`: redireciona o backup para arquivo no **servidor host**;
- `$(date +%F-%H%M)`: inclui data e horário no nome do arquivo;
- `sudo bash -c`: permite criar o arquivo dentro da pasta protegida.

### 18.4 Verificar se o arquivo existe

```bash
sudo ls -lh /var/backups/zabbix/
```

Para verificar a estrutura de um dump sem restaurá-lo, substitua `NOME_DO_BACKUP.dump`:

```bash
sudo bash -c 'docker compose exec -T postgres pg_restore -l \
  < /var/backups/zabbix/NOME_DO_BACKUP.dump'
```

**Explicação:** o redirecionamento `<` lê o arquivo no host; como a pasta tem acesso restrito, todo o comando é executado em um shell root.

> **Atenção:** substituir `NOME_DO_BACKUP.dump` pelo nome real. Verificar a lista do dump não substitui um **teste de restauração** em ambiente separado.

### 18.5 Restaurar — procedimento de referência (operação destrutiva)

**Prefira restaurar em um ambiente de testes isolado.** Uma restauração sobre a base existente pode substituir configurações e métricas atuais. Planeje indisponibilidade e confirme que o arquivo é válido antes de continuar.

Se a restauração **sobre a mesma base** for indispensável:

1. Faça um novo backup do estado atual.
2. Pare as aplicações que acessam o banco, **mantendo o PostgreSQL ativo**:

   ```bash
   cd /opt/zabbix-docker
   sudo docker compose stop zabbix-server zabbix-web
   ```

3. Restaure o arquivo, revisando os efeitos de `--clean` e `--if-exists`:

   ```bash
   sudo bash -c 'docker compose exec -T postgres \
     pg_restore -U zabbix -d zabbix \
     --clean --if-exists --single-transaction --exit-on-error \
     < /var/backups/zabbix/NOME_DO_BACKUP.dump'
   ```

4. Se a restauração terminou **sem erros**, inicie as aplicações:

   ```bash
   sudo docker compose start zabbix-server zabbix-web
   sudo docker compose logs --tail=100 zabbix-server
   ```

**Advertência:** a restauração pressupõe um dump consistente do banco `zabbix`, permissões adequadas e versões compatíveis. Caso ocorra erro, **não execute repetidamente às cegas**: preserve os logs e investigue a incompatibilidade.

### 18.6 Estratégia mínima recomendada

- Backup diário do banco com retenção definida.
- Cópia em **outro servidor ou armazenamento externo** protegido.
- Restrição de acesso aos arquivos `.dump`.
- Teste periódico de restauração em outro ambiente.
- Backup adicional do `compose.yaml`, `.env` (em armazenamento seguro) e documentação de customizações.

## 19. Atualizações e manutenção

### 19.1 Atualizar as imagens na mesma linha do Zabbix 8.0

Antes de atualizar:

1. Leia as notas de versão e registre os componentes atuais.
2. Faça backup do banco.
3. Reserve uma janela de manutenção para ambiente de produção.

Depois:

```bash
cd /opt/zabbix-docker
sudo docker compose ps
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
```

**Explicação:** `pull` obtém versões mais recentes das imagens declaradas; `up -d` recria os containers que precisarem de atualização. O volume do PostgreSQL permanece, desde que você não remova volumes explicitamente.

**Importante:** a imagem `postgres:17-alpine` acompanha atualizações da linha 17. Atualizar entre **versões principais** de PostgreSQL (por exemplo, 17 → 18) exige migração planejada e não deve ser feito apenas trocando o número da imagem.

### 19.2 Atualizações automáticas

Este pacote **não ativa atualizações automáticas de imagens**. Isso é proposital: em monitoramento corporativo, novas imagens podem exigir testes, backup e janela de mudança.

### 19.3 Reiniciar o servidor Linux

```bash
sudo reboot
```

Após o boot, valide:

```bash
sudo systemctl status docker --no-pager
cd /opt/zabbix-docker
sudo docker compose ps
```

Os serviços configurados com `restart: unless-stopped` são normalmente reiniciados após o boot, desde que o Docker esteja ativo e eles não tenham sido parados manualmente.

## 20. Solução de problemas

### Erro: `docker: command not found`

**Causa comum:** Engine/CLI não instalados ou instalação incompleta. Volte às etapas 6 (Ubuntu) ou 7 (Debian).

### Erro: `docker: 'compose' is not a docker command`

**Causa comum:** plugin Compose ausente.

```bash
sudo apt install -y docker-compose-plugin
sudo docker compose version
```

Se o APT não localizar o pacote, verifique se o repositório **oficial da Docker** foi configurado.

### Erro: `permission denied ... /var/run/docker.sock`

Execute os comandos com `sudo`. Se o erro persistir, verifique:

```bash
sudo systemctl status docker --no-pager
```

### Erro: `Cannot connect to the Docker daemon`

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

### Erro: `port is already allocated` ou `bind: address already in use`

Uma aplicação já ocupa `8085` ou `10051`. Identifique as portas:

```bash
sudo ss -lntp | grep -E ':(8085|10051)\b' || true
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Se a ocupada for **8085**, ajuste `ZABBIX_WEB_PORT=8090` em `.env` e execute `sudo docker compose up -d`. Se for **10051**, investigue o outro serviço antes de alterar o mapeamento no `compose.yaml` — agentes/proxies também precisam conhecer a porta escolhida.

### Erro: página não carrega pelo navegador

1. Confirme se o serviço web foi criado: `sudo docker compose ps`.
2. Veja os logs: `sudo docker compose logs --tail=150 zabbix-web`.
3. Teste local: `curl -I http://127.0.0.1:8085`.
4. Confira se o IP usado é o correto: `hostname -I`.
5. Verifique firewall externo/VLAN/roteamento e se o cliente consegue chegar ao IP e à porta.

### Erro: Zabbix Server não conecta ao PostgreSQL

```bash
sudo docker compose ps
sudo docker compose logs --tail=150 postgres
sudo docker compose logs --tail=150 zabbix-server
sudo docker compose exec -T postgres pg_isready -U zabbix -d zabbix
```

**Causas possíveis:** banco ainda inicializando, volume de instalação anterior com senha diferente, falta de espaço em disco ou erros de permissão/arquivo.

**Nunca resolva problema de senha com `docker compose down -v` em um banco que contém dados importantes.** Se o volume já existia, a senha de inicialização do container PostgreSQL não altera retroativamente o usuário do banco.

### Erro: `no configuration file provided` ou `compose.yaml` não encontrado

Entre na pasta correta:

```bash
cd /opt/zabbix-docker
ls -la
sudo docker compose config --quiet
```

### Erro: `invalid interpolation format` ou YAML inválido

Valide:

```bash
sudo docker compose config --quiet
```

Confira indentação no `compose.yaml` (YAML usa espaços, não tabulação) e eventuais caracteres especiais no `.env`.

### Erro: `no space left on device`

```bash
df -h
sudo docker system df
sudo docker volume ls
```

**Não remova volumes indiscriminadamente.** Investigue imagens sem uso, logs, backups e crescimento do banco. Reduza retenção de dados pelo Zabbix conforme a necessidade.

### Erro: o Zabbix não apresenta dados dos hosts

- Confirme que o host foi cadastrado e está habilitado.
- Verifique se há template e itens associados.
- Para agentes ativos, confira `ServerActive` e hostname configurados no agente.
- Para agentes passivos, confira porta `10050/TCP`, firewall e o IP de origem visto pelo agente.
- Para SNMP, valide SNMPv3, credenciais, endereço e reachability.
- Confira horários/NTP, **Dados recentes** e logs do agente.

## 21. Checklist e próximos passos

Marque os itens após concluir:

- [ ] Versão do Ubuntu/Debian e IP do servidor identificados.
- [ ] Docker Engine instalado e ativo.
- [ ] Docker Compose v2 disponível.
- [ ] `hello-world` executado com sucesso.
- [ ] ZIP extraído em `/opt/zabbix-docker`.
- [ ] `.env` criado com senha forte e permissão `600`.
- [ ] `docker compose config --quiet` sem erros.
- [ ] Containers `postgres`, `zabbix-server` e `zabbix-web` ativos.
- [ ] PostgreSQL em estado saudável e `pg_isready` aprovado.
- [ ] Painel acessível em `http://IP_DO_SERVIDOR:8085`.
- [ ] Senha `Admin` alterada.
- [ ] Acesso de rede limitado e plano para HTTPS definido.
- [ ] Primeiro host cadastrado e apresentando dados.
- [ ] Backup criado, copiado para outro local e restauração testada.

### Próxima evolução sugerida

Depois de dominar a implantação básica, implemente **Zabbix Agent 2 em servidores Linux/Windows**, monitoramento **SNMPv3** de switches e roteadores, notificações de incidentes, dashboards e regras de retenção. Caso a instância seja essencial à operação, planeje monitoramento do próprio host Docker, backup externo e recuperação de desastres.

## 22. Referências oficiais

- Docker Engine no Ubuntu: <https://docs.docker.com/engine/install/ubuntu/>
- Docker Engine no Debian: <https://docs.docker.com/engine/install/debian/>
- Docker Compose: <https://docs.docker.com/compose/>
- Referência do arquivo Compose: <https://docs.docker.com/reference/compose-file/>
- Segurança e firewall do Docker: <https://docs.docker.com/engine/network/packet-filtering-firewalls/>
- Documentação de containers Zabbix 8.0: <https://www.zabbix.com/documentation/8.0/pt/manual/installation/install/containers>
- Requisitos Zabbix 8.0: <https://www.zabbix.com/documentation/8.0/pt/manual/installation/requirements>
- Login inicial do Zabbix: <https://www.zabbix.com/documentation/8.0/pt/manual/installation/guides/basic_config/login>
- Notas de versão: <https://www.zabbix.com/release_notes>
- Imagem oficial do Zabbix Server (PostgreSQL): <https://hub.docker.com/r/zabbix/zabbix-server-pgsql>
- Imagem oficial do Zabbix Web (Nginx + PostgreSQL): <https://hub.docker.com/r/zabbix/zabbix-web-nginx-pgsql>

---

**Resumo da implantação:** instalar Docker Engine + Compose → extrair o pacote → executar `sudo bash instalar.sh` → validar containers/logs → abrir `http://IP:8085` → trocar a senha padrão → cadastrar hosts → configurar backups e segurança.
