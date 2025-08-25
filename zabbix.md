# Zabbix no Ubuntu 24 com Docker

Este repositório contém um guia para instalar e configurar o Zabbix 7.4 no Ubuntu 22 utilizando Docker e Docker Compose. 
As configurações de banco de dados e web são gerenciadas através de variáveis de ambiente para maior segurança e flexibilidade.

A decisão de colcoar no Ubuntu 22 foi devido a incompatibilidade do Zabbix Proxy 7.4 no Debian 12.

## Pré-requisitos

- Ubuntu 22 LTS
- Acesso `sudo`
- Conexão com a internet

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

## Contribuição

Sinta-se à vontade para contribuir com melhorias ou correções. Abra uma *issue* ou *pull request* no GitHub.

## Licença

Este projeto está licenciado sob a [Licença MIT](https://opensource.org/licenses/MIT).

