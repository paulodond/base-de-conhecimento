# Tutorial de Implantação GLPI + Grafana com Docker, Nginx e SSL

Este tutorial descreve como configurar o GLPI e o Grafana usando Docker, Nginx como proxy reverso e SSL com Let's Encrypt. Os dados sensíveis foram substituídos por placeholders (ex.: `SUA_SENHA`, `SEU_DOMINIO`) para personalização.

## 1. Preparação do Sistema

### 1.1. Atualização do Sistema e Instalação de Dependências Básicas

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release nginx certbot python3-certbot-nginx
```

### 1.2. Instalação do Docker e Docker Compose

```bash
# Atualizar pacotes e instalar dependências
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Adicionar chave GPG oficial do Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Adicionar repositório Docker para Debian Bookworm
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Atualizar fontes e instalar Docker e Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Adicionar usuário atual ao grupo docker
sudo usermod -aG docker $USER
```

> **Nota**: Faça logout e login novamente para aplicar as permissões do grupo `docker`.

## 2. Configuração do Docker Compose

### 2.1. Criar Estrutura de Diretórios

```bash
mkdir -p /docker/glpi-server
cd /docker/glpi-server
```

### 2.2. Criar arquivo `.env` para variáveis sensíveis

Crie um arquivo `.env` com as variáveis necessárias:

```bash
vim .env
```

Adicione o seguinte conteúdo, substituindo os placeholders pelos valores desejados:

```bash
# Config MariaDB
MYSQL_ROOT_PASSWORD=SUA_SENHA_MARIADB_ROOT
MYSQL_DATABASE=glpidb
MYSQL_USER=glpi
MYSQL_PASSWORD=SUA_SENHA_MARIADB_USER

# Config GLPI
GLPI_LANG=pt_BR
MARIADB_HOST=glpi-mariadb
MARIADB_DATABASE=glpidb
MARIADB_USER=glpi
MARIADB_PASSWORD=SUA_SENHA_MARIADB_USER

# Config Grafana
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=SUA_SENHA_GRAFANA_ADMIN
GF_SERVER_ROOT_URL=https://SEU_DOMINIO:3000
```

> **Nota**: Substitua `SUA_SENHA_MARIADB_ROOT`, `SUA_SENHA_MARIADB_USER`, `SUA_SENHA_GRAFANA_ADMIN` e `SEU_DOMINIO` pelos valores apropriados. Use senhas fortes e únicas.

Restringir permissões do arquivo `.env`:

```bash
chmod 600 .env
```

### 2.3. Criar arquivo `docker-compose.yml`

```bash
vim docker-compose.yml
```

Adicione o seguinte conteúdo:

```yaml
services:
  glpi-mariadb:
    image: mariadb:10.7
    container_name: glpi-mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ./mariadb-data:/var/lib/mysql
    networks:
      - glpi-network

  glpi-app:
    image: diouxx/glpi:latest
    container_name: glpi-app
    restart: always
    environment:
      GLPI_LANG: ${GLPI_LANG}
      MARIADB_HOST: ${MARIADB_HOST}
      MARIADB_DATABASE: ${MARIADB_DATABASE}
      MARIADB_USER: ${MARIADB_USER}
      MARIADB_PASSWORD: ${MARIADB_PASSWORD}
    volumes:
      - ./glpi-data:/var/www/html/glpi
      - ./glpi-config:/var/www/html/glpi/config
      - ./glpi-files:/var/www/html/glpi/files
      - ./glpi-plugins:/var/www/html/glpi/plugins
    expose:
      - "80"
    depends_on:
      - glpi-mariadb
    networks:
      - glpi-network
    ports:
      - "SEU_IP:8080:80"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    expose:
      - "3000"
    volumes:
      - ./grafana-data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_USER: ${GF_SECURITY_ADMIN_USER}
      GF_SECURITY_ADMIN_PASSWORD: ${GF_SECURITY_ADMIN_PASSWORD}
      GF_SERVER_ROOT_URL: ${GF_SERVER_ROOT_URL}
      GF_SERVER_SERVE_FROM_SUB_PATH: "true"
      GF_SECURITY_ALLOW_EMBEDDING: "true"
    depends_on:
      - glpi-mariadb
    networks:
      - glpi-network
    ports:
      - "SEU_IP:3001:3000"

networks:
  glpi-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  mariadb-data:
  glpi-data:
  glpi-config:
  glpi-files:
  glpi-plugins:
  grafana-data:
```

> **Nota**: Substitua `SEU_IP` pelo IP do seu servidor ou deixe em branco para usar `localhost`.

### 2.4. Criar diretórios de persistência e ajustar permissões

```bash
mkdir -p mariadb-data glpi-data glpi-config glpi-files glpi-plugins grafana-data
sudo chown -R 472:472 grafana-data
```

### 2.5. Inicializar os containers

```bash
docker compose up -d
docker compose ps
docker compose logs -f
```

## 3. Configuração do Nginx como Proxy Reverso

### 3.1. Backup da configuração original

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.backup
```

### 3.2. Ajustar configuração principal do Nginx

Edite o arquivo `/etc/nginx/nginx.conf`:

```bash
vim /etc/nginx/nginx.conf
```

Substitua o conteúdo pelo seguinte:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
    multi_accept on;
    use epoll;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
    ssl_session_timeout 10m;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    proxy_buffering on;
    proxy_buffer_size 16k;
    proxy_buffers 8 16k;
    proxy_busy_buffers_size 24k;
    proxy_temp_file_write_size 128k;
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;

    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log;

    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### 3.3. Configurar proxy reverso para GLPI e Grafana

#### 3.3.1. Configuração temporária para validação Let's Encrypt (porta 80)

Crie o arquivo `/etc/nginx/sites-available/temp-glpi`:

```bash
vim /etc/nginx/sites-available/temp-glpi
```

Adicione:

```nginx
server {
    listen 80;
    server_name SEU_DOMINIO;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$server_name:8081$request_uri;
    }
}
```

Ative o site e prepare diretórios para o Certbot:

```bash
sudo ln -s /etc/nginx/sites-available/temp-glpi /etc/nginx/sites-enabled/
sudo mkdir -p /var/www/html/.well-known/acme-challenge
sudo chown -R www-data:www-data /var/www/html
sudo nginx -t
sudo systemctl reload nginx
```

### 3.4. Obter certificados SSL com Certbot

```bash
sudo certbot certonly --webroot \
  --webroot-path=/var/www/html \
  --email SEU_EMAIL \
  --agree-tos \
  --no-eff-email \
  -d SEU_DOMINIO
```

> **Nota**: Substitua `SEU_EMAIL` e `SEU_DOMINIO` pelos valores apropriados.

### 3.5. Configurar Virtual Hosts Nginx com SSL

#### 3.5.1. GLPI (porta 8081)

Crie `/etc/nginx/sites-available/glpi-SEU_DOMINIO`:

```bash
vim /etc/nginx/sites-available/glpi-SEU_DOMINIO
```

Adicione:

```nginx
server {
    listen 8081 ssl http2;
    server_name SEU_DOMINIO;

    ssl_certificate /etc/letsencrypt/live/SEU_DOMINIO/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/SEU_DOMINIO/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    client_max_body_size 100M;

    proxy_set_header Host $host:$server_port;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port $server_port;

    location / {
        proxy_pass http://SEU_IP:8080;
        proxy_redirect off;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        proxy_pass http://SEU_IP:8080;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    access_log /var/log/nginx/glpi-access.log main;
    error_log /var/log/nginx/glpi-error.log;
}

server {
    listen 8081;
    server_name SEU_DOMINIO;
    return 301 https://$server_name:$server_port$request_uri;
}
```

> **Nota**: Substitua `SEU_DOMINIO` e `SEU_IP` pelos valores apropriados.
> As portas usadas aqui froam para atende ao meu cenário devio ao uso da porta 80 e 443 por outros serviço.

#### 3.5.2. Grafana (porta 3000)

Crie `/etc/nginx/sites-available/grafana-SEU_DOMINIO`:

```bash
vim /etc/nginx/sites-available/grafana-SEU_DOMINIO
```
> **Nota**: Substitua `SEU_DOMINIO` e `SEU_IP` pelos valores apropriados.

Adicione:

```nginx
server {
    listen 3000 ssl http2;
    server_name SEU_DOMINIO;

    ssl_certificate /etc/letsencrypt/live/SEU_DOMINIO/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/SEU_DOMINIO/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    proxy_set_header Host $host:$server_port;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_set_header X-Forwarded-Host $host:$server_port;

    location / {
        proxy_pass http://SEU_IP:3001;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    location /api/ {
        proxy_pass http://SEU_IP:3001;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        proxy_pass http://SEU_IP:3001;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    access_log /var/log/nginx/grafana-access.log main;
    error_log /var/log/nginx/grafana-error.log;
}

server {
    listen 3000;
    server_name SEU_DOMINIO;
    return 301 https://$server_name:$server_port$request_uri;
}
```

### 3.6. Ativar sites Nginx e remover padrão

```bash
sudo ln -s /etc/nginx/sites-available/glpi-SEU_DOMINIO /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/grafana-SEU_DOMINIO /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
sudo nginx -t
```

### 3.7. Reiniciar containers e Nginx

```bash
docker compose down
docker compose up -d
sudo systemctl reload nginx
```

Agora você tem o GLPI e o Grafana rodando com Docker + Nginx + SSL 🎊.
Basta acessar:
🌐 GLPI: https://seu-dominio.com:8081

📊 Grafana: https://seu-dominio.com:3000

OBs: O SQL Server é o nome do container que você atribuiu lá no docker-compose.yml  e não localhost

<img width="931" height="562" alt="image" src="https://github.com/user-attachments/assets/d7bf801f-cd24-40f2-8afb-062e6b075a61" />


