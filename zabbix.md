# Zabbix + Zabbix Proxy no Ubuntu 22 com Docker

Este repositório contém um guia para instalar e configurar o Zabbix 7.4 no Ubuntu 22 utilizando Docker e Docker Compose. 
As configurações de banco de dados e web são gerenciadas através de variáveis de ambiente para maior segurança e flexibilidade.

## Implementação

- Ubuntu 22 LTS
- Zabbix 7.4
- Zabbix Proxy 7.4

## Estrutura do Projeto

```
/docker/zabbix-server/
├── .env
└── docker-compose.yml
/docker/zabbix-snmptraps/
└── snmptraps.log
/docker/zabbix-mibs/
└── (arquivos MIBs)
```

## Instalação

Siga os passos abaixo para configurar o Zabbix no seu sistema.

### 1. Atualizar o Sistema e Instalar Dependências

Primeiro, atualize os pacotes do sistema e instale as dependências necessárias para o Docker:

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common \
    gnupg \
    lsb-release
```

### 2. Instalar Docker Engine

Adicione a chave GPG oficial do Docker e configure o repositório. Em seguida, instale o Docker Engine, CLI, containerd e os plugins necessários:

```bash
# Adicionar chave GPG do Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Configurar repositório Docker para Ubuntu 24.04 (lunar)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Atualizar lista de pacotes
sudo apt update

# Instalar Docker e plugins
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3. Criar Diretórios Base para Zabbix

Crie os diretórios necessários para os arquivos de configuração e dados do Zabbix, e ajuste as permissões:

```bash
sudo mkdir -p /docker/zabbix-server
sudo chown $USER:$USER /docker/zabbix-server

mkdir -p /docker/zabbix-snmptraps
touch /docker/zabbix-snmptraps/snmptraps.log
chown 200:200 /docker/zabbix-snmptraps/snmptraps.log
chmod 664 /docker/zabbix-snmptraps/snmptraps.log

sudo mkdir -p /docker/zabbix-mibs
sudo chown -R 1000:1000 /docker/zabbix-mibs
sudo chmod -R 755 /docker/zabbix-mibs

# Copiar as MIBs (se necessário)
cp -r /usr/share/snmp/mibs/* /docker/zabbix-mibs/
```

### 4. Configurar Variáveis de Ambiente (.env)

Navegue até o diretório `/docker/zabbix-server` e crie o arquivo `.env` com as seguintes variáveis. **Lembre-se de alterar os valores padrão para senhas fortes e configurações adequadas ao seu ambiente de produção.**

```bash
cd /docker/zabbix-server/

cat > .env << 'EOL'
# Zabbix PostgreSQL Configuration
POSTGRES_USER=your_user              # Altere conforme suas configurações
POSTGRES_PASSWORD=your_password          # Altere para senha forte antes de usar em produção
POSTGRES_DB=your_db                # Nome do banco

# Zabbix Server Configuration
ZBX_TIMEOUT=30

# Zabbix Web Configuration
PHP_TZ=your_timezone          # Ajuste seu timezone

# Ports Configuration
ZABBIX_SERVER_PORT=10051
ZABBIX_WEB_PORT=8083
ZABBIX_AGENT_PORT=10050
EOL

# Restringir acesso ao arquivo .env
chmod 600 .env
```

### 5. Configurar Docker Compose (docker-compose.yml)

No mesmo diretório `/docker/zabbix-server`, crie o arquivo `docker-compose.yml` com a seguinte configuração. Este arquivo define os serviços do Zabbix (PostgreSQL, Server, Web e SNMP Traps) e suas interconexões.

```yaml
services:
  zabbix-postgres:
    image: postgres:16
    container_name: zabbix-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - zabbix-network

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:ubuntu-7.4-latest
    container_name: zabbix-server
    hostname: your_hostname
    restart: unless-stopped
    environment:
      DB_SERVER_HOST: zabbix-postgres
      DB_SERVER_PORT: 5432
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      ZBX_TIMEOUT: ${ZBX_TIMEOUT}
      ZBX_HOSTNAME: your_hostname
      ZBX_ENABLE_SNMP_TRAPS: "true"
    ports:
      - "${ZABBIX_SERVER_PORT}:10051"
    depends_on:
      zabbix-postgres:
        condition: service_healthy
    volumes:
      - /docker/zabbix-snmptraps:/var/lib/zabbix/snmptraps
      - /docker/zabbix-mibs:/var/lib/zabbix/mibs
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - zabbix-network

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-7.4-latest
    container_name: zabbix-web
    restart: unless-stopped
    environment:
      DB_SERVER_HOST: zabbix-postgres
      DB_SERVER_PORT: 5432
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      ZBX_SERVER_HOST: zabbix-server
      ZBX_SERVER_PORT: 10051
      PHP_TZ: ${PHP_TZ}
      ZBX_SERVER_NAME: your_server_name
    ports:
      - "${ZABBIX_WEB_PORT}:8080"
    depends_on:
      zabbix-postgres:
        condition: service_healthy
      zabbix-server:
        condition: service_started
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - zabbix-network

  zabbix-snmptraps:
    image: zabbix/zabbix-snmptraps:ubuntu-7.4-latest
    container_name: zabbix-snmptraps
    restart: unless-stopped
    volumes:
      - /docker/zabbix-snmptraps:/var/lib/zabbix/snmptraps
      - /docker/zabbix-mibs:/var/lib/zabbix/mibs
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - zabbix-network

networks:
  zabbix-network:
    driver: bridge
```

### 6. Criar Diretórios para Volumes

Crie os diretórios para os volumes de dados do PostgreSQL e SNMP traps, e ajuste as permissões:

```bash
mkdir -p /docker/zabbix-server/postgres-data
mkdir -p /docker/zabbix-server/snmptraps

# Ajustar permissões para snmptraps (UID e GID do usuário zabbix dentro do container)
sudo chown 115:122 /docker/zabbix-server/snmptraps
sudo chmod 770 /docker/zabbix-server/snmptraps
```

### 7. Inicializar Containers

Finalmente, inicie os serviços do Zabbix usando Docker Compose:

```bash
cd /docker/zabbix-server
docker compose up -d
```

Após a inicialização, o Zabbix estará acessível na porta `8083` do seu servidor (ou a porta que você configurou em `ZABBIX_WEB_PORT` no arquivo `.env`).


# Zabbix Proxy com SQLite3

Este repositório contém um guia para instalar e configurar um Zabbix Proxy utilizando SQLite3 no Ubuntu 22.04. Este proxy é ideal para ambientes menores ou para testar a funcionalidade do Zabbix Proxy sem a necessidade de um banco de dados externo.

## Pré-requisitos

- Ubuntu 22.04 LTS
- Acesso `sudo`
- Conexão com a internet
- Zabbix Server já instalado e configurado (em container ou em máquina virtual/física).

## Instalação

Siga os passos abaixo para configurar o Zabbix Proxy no seu sistema.

### 1. Baixar e Instalar o Pacote de Release do Zabbix

Baixe e instale o pacote de release do Zabbix para o Ubuntu 22.04. Certifique-se de que a versão do Zabbix (neste exemplo, 7.2) seja compatível com a versão do seu Zabbix Server.

```bash
wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu22.04_all.deb
dpkg -i zabbix-release_latest+ubuntu22.04_all.deb
```

### 2. Atualizar Lista de Pacotes

Atualize a lista de pacotes do sistema para incluir os novos repositórios do Zabbix:

```bash
sudo apt update
```

### 3. Instalar Zabbix Proxy com SQLite3

Instale o pacote `zabbix-proxy-sqlite3`. Esta opção é mais simples, pois utiliza um banco de dados SQLite local, eliminando a necessidade de configurar um banco de dados externo como PostgreSQL ou MySQL.

```bash
sudo apt install -y zabbix-proxy-sqlite3
```

### 4. Backup do Arquivo de Configuração Original

É uma boa prática fazer um backup do arquivo de configuração original antes de realizar qualquer alteração:

```bash
sudo cp /etc/zabbix/zabbix_proxy.conf /etc/zabbix/zabbix_proxy.conf.bak
```

### 5. Editar o Arquivo de Configuração do Zabbix Proxy

Edite o arquivo `/etc/zabbix/zabbix_proxy.conf` para ajustar os parâmetros de conexão com o Zabbix Server e outras configurações. Utilize um editor de texto de sua preferência (ex: `vim.tiny`, `nano`).

```bash
vim.tiny /etc/zabbix/zabbix_proxy.conf
```

**Parâmetros que devem ser ajustados:**

- `Server=your_zabbix_server_ip`: Defina o endereço IP ou hostname do seu Zabbix Server. Se o Zabbix Server estiver rodando em um container, use o IP do container.
- `Hostname=your_proxy_hostname`: Escolha um nome único para o seu proxy. Este nome deve ser exatamente o mesmo que você registrará no frontend do Zabbix Server.
- `ListenPort=your_chosen_port`: A porta padrão para o Zabbix Server é 10051. Para evitar conflitos, é recomendável usar uma porta diferente para o proxy, como `10061`. Esta porta será usada pelos agentes que se conectarão a este proxy.
- `DBName=/tmp/zabbix_proxy`: Coloque esse caminha exatamente`.

Exemplo de configuração (apenas os parâmetros alterados):

```ini
Server=your_zabbix_server_ip
Hostname=your_proxy_hostname
ListenPort=your_chosen_port
DBName=your_database_path
```

### 6. Habilitar e Iniciar o Serviço do Proxy

Após configurar o arquivo `zabbix_proxy.conf`, habilite e inicie o serviço do Zabbix Proxy:

```bash
sudo systemctl enable --now zabbix-proxy
```

### 7. Verificar o Status do Serviço

Para confirmar que o serviço do Zabbix Proxy está rodando corretamente, verifique seu status:

```bash
sudo systemctl status zabbix-proxy --no-pager
```

Você deverá ver uma saída indicando que o serviço está `active (running)`.

## Configuração no Zabbix Frontend

Após a instalação e configuração do proxy, você precisará registrá-lo no frontend do Zabbix Server. Navegue até:

`Administração` -> `Proxies` -> `Criar proxy`

- **Nome do Proxy:** Insira o `Hostname` que você definiu no arquivo `zabbix_proxy.conf`.
- **Modo de Proxy:** Selecione `Ativo` ou `Passivo` conforme sua necessidade (o modo passivo é o padrão para a maioria das instalações).

## Contribuição

Sinta-se à vontade para contribuir com melhorias ou correções. Abra uma *issue* ou *pull request* no GitHub.

## Licença

Este projeto está licenciado sob a [Licença MIT](https://opensource.org/licenses/MIT).




## Contribuição

Sinta-se à vontade para contribuir com melhorias ou correções. Abra uma *issue* ou *pull request* no GitHub.

## Licença

Este projeto está licenciado sob a [Licença MIT](https://opensource.org/licenses/MIT).

