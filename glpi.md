````markdown
# 📘 Tutorial de Implantação GLPI + Grafana com Docker, Nginx e SSL

Este guia mostra como implantar o **GLPI** e o **Grafana** em containers Docker, utilizando **Nginx como proxy reverso** e
**Certbot para SSL** 🔐.  
Tudo foi deixado **genérico**, sem credenciais ou domínios reais. Basta **editar os arquivos `.env` e as configs do Nginx**
para seu ambiente.

---

## 🛠️ 1. Preparação do Sistema

### 🔄 1.1. Atualizar sistema e instalar dependências

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release nginx certbot python3-certbot-nginx
````

### 🐳 1.2. Instalar Docker e Docker Compose

```bash
# Instalar dependências
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Adicionar chave GPG do Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Adicionar repositório Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker e plugins
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Adicionar usuário ao grupo docker
sudo usermod -aG docker $USER
```

---

## 📂 2. Configuração do Docker Compose

### 📁 2.1. Criar estrutura de diretórios

```bash
mkdir -p /docker/glpi-server
cd /docker/glpi-server
```

### 🔑 2.2. Criar arquivo `.env` com variáveis

vim.tiny .env

```bash
cat > .env << 'EOL'
# Config MariaDB
MYSQL_ROOT_PASSWORD=defina_sua_senha
MYSQL_DATABASE=glpidb
MYSQL_USER=glpi
MYSQL_PASSWORD=defina_sua_senha

# Config GLPI
GLPI_LANG=pt_BR
MARIADB_HOST=glpi-mariadb
MARIADB_DATABASE=glpidb
MARIADB_USER=glpi
MARIADB_PASSWORD=defina_sua_senha

# Config Grafana
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=defina_sua_senha
GF_SERVER_ROOT_URL=https://seu-dominio.com:3000
EOL

# Restringido persmissões ao .env
chmod 600 .env
```

⚠️ **Importante:** Altere as senhas e o domínio antes de usar.

---

### 📜 2.3. Criar arquivo `docker-compose.yml`

vim.tiny docker-compose.yml

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

---

### 📂 2.4. Criar diretórios de persistência

```bash
mkdir -p mariadb-data glpi-data glpi-config glpi-files glpi-plugins grafana-data
sudo chown -R 472:472 grafana-data
```

### ▶️ 2.5. Inicializar containers

```bash
docker compose up -d
docker compose ps
docker compose logs -f
```

---

## 🌐 3. Configuração do Nginx como Proxy Reverso

### 💾 3.1. Backup da configuração

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.backup
```

### ⚙️ 3.2. Ajustar `nginx.conf`

* Configure segurança (TLS 1.2/1.3, cabeçalhos de segurança, gzip etc.)
* Inclua diretivas para **proxy reverso** e **logs**

```
vim.tiny /etc/nginx/nginx.conf

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


---

## 🔐 4. Certificados SSL com Certbot

```bash
sudo certbot certonly --webroot \
  --webroot-path=/var/www/html \
  --email seu-email@dominio.com \
  --agree-tos \
  --no-eff-email \
  -d seu-dominio.com
```

---

## 📑 5. Virtual Hosts no Nginx

Crie os arquivos:

* `/etc/nginx/sites-available/glpi-seu-dominio.com`
* `/etc/nginx/sites-available/grafana-seu-dominio.com`

👉 Edite para apontar para o **IP do servidor Docker** (exemplo: `172.20.0.2`) e para seu **domínio**.

---

## ✅ 6. Ativar configurações e reiniciar

```bash
sudo ln -s /etc/nginx/sites-available/glpi-seu-dominio.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/grafana-seu-dominio.com /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
sudo nginx -t

docker compose down
docker compose up -d
sudo systemctl reload nginx
```

---

## 🎉 Conclusão

Agora você tem o **GLPI** e o **Grafana** rodando com **Docker + Nginx + SSL** 🎊.
Basta acessar:

* 🌐 **GLPI:** `https://seu-dominio.com:8081`
* 📊 **Grafana:** `https://seu-dominio.com:3000`

---

```

---

Quer que eu já prepare também o **repositório inicial** (com estrutura `/docker/glpi-server/`, `.gitignore`, `README.md` e placeholders dos arquivos `.env` e `docker-compose.yml`) para você só dar um `git init` e publicar?
```
