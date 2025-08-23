````markdown
# 🚀 Guia Completo: Servidor RustDesk com Docker, Nginx e SSL

Aprenda a configurar seu próprio servidor **RustDesk** de forma segura, usando Docker, Nginx e Certbot.  
Ideal para controle total de acesso remoto e autenticação.

---

## 📖 Sumário

1. [Instalação do Docker e Docker Compose](#1-instalação-do-docker-e-docker-compose)  
2. [Geração de Chaves Persistentes](#2-geração-de-chaves-persistentes)  
3. [Configuração com Docker Compose](#3-configuração-com-docker-compose)  
4. [Configuração do Nginx](#4-configuração-do-nginx)  
5. [Configurar SSL com Certbot](#5-configurar-ssl-com-certbot)  
6. [Configuração do Cliente RustDesk](#6-configuração-do-cliente-rustdesk)  
7. [Segurança Adicional](#7-segurança-adicional)  

---

## 1️⃣ Instalação do Docker e Docker Compose

Atualize o sistema e instale dependências:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
````

🔑 **Adicionar chave GPG oficial do Docker**:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

📦 **Adicionar repositório Docker**:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verifique as versões instaladas:

```bash
docker --version
docker compose version
```

💡 **Dica:** Permitir Docker sem `sudo`:

```bash
sudo usermod -aG docker $USER
```

---

## 2️⃣ Geração de Chaves Persistentes

Crie diretório para armazenar chaves:

```bash
sudo mkdir -p /opt/rustdesk-keys
```

📦 **Gerar chaves usando container temporário**:

```bash
sudo docker run --name rustdesk-temp -d -p 21115-21119:21115-21119 rustdesk/rustdesk-server hbbs
sudo docker cp rustdesk-temp:/root/id_ed25519 /opt/rustdesk-keys/
sudo docker cp rustdesk-temp:/root/id_ed25519.pub /opt/rustdesk-keys/
sudo docker stop rustdesk-temp
sudo docker rm rustdesk-temp
```

🔑 **Agora você tem:**

* `/opt/rustdesk-keys/id_ed25519` → chave privada
* `/opt/rustdesk-keys/id_ed25519.pub` → chave pública

🖼️ *Sugestão de imagem:* ilustração de chaves SSH com ícones de segurança.

---

## 3️⃣ Configuração com Docker Compose

Crie diretório e arquivo:

```bash
sudo mkdir -p /opt/rustdesk-server
cd /opt/rustdesk-server
nano docker-compose.yml
```

Exemplo de `docker-compose.yml`:

```yaml
services:
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server
    command: hbbs -r SEU_DOMINIO:21117 -k _
    volumes:
      - ./data:/root
      - /opt/rustdesk-keys/id_ed25519:/root/id_ed25519:ro
      - /opt/rustdesk-keys/id_ed25519.pub:/root/id_ed25519.pub:ro
    ports:
      - "21115:21115"
      - "21116:21116"
      - "21116:21116/udp"
    restart: unless-stopped
    user: "1000:1000"
    read_only: true
    security_opt:
      - no-new-privileges:true

  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server
    command: hbbr
    volumes:
      - ./data:/root
      - /opt/rustdesk-keys/id_ed25519:/root/id_ed25519:ro
      - /opt/rustdesk-keys/id_ed25519.pub:/root/id_ed25519.pub:ro
    ports:
      - "21117:21117"
      - "21118:21118"
      - "21119:21119"
    restart: unless-stopped
    user: "1000:1000"
    read_only: true
    security_opt:
      - no-new-privileges:true
```

Suba os serviços:

```bash
sudo docker compose up -d
sudo docker compose ps
```

🖼️ *Sugestão de imagem:* esquema visual de containers Docker comunicando entre si.

---

## 4️⃣ Configuração do Nginx

Instalar Nginx:

```bash
sudo apt install -y nginx
```

Criar configuração (substitua `SEU_DOMINIO`):

```nginx
server {
    listen 80;
    server_name SEU_DOMINIO;

    location / {
        proxy_pass http://127.0.0.1:21117;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Ativar site:

```bash
sudo ln -s /etc/nginx/sites-available/SEU_DOMINIO /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

🖼️ *Sugestão de imagem:* diagrama Nginx → RustDesk.

---

## 5️⃣ Configurar SSL com Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d SEU_DOMINIO
sudo certbot renew --dry-run
```

Se usar porta HTTPS customizada (ex.: 10447):

```nginx
server {
    listen 10447 ssl http2;
    server_name SEU_DOMINIO;

    ssl_certificate /etc/letsencrypt/live/SEU_DOMINIO/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/SEU_DOMINIO/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    add_header Content-Security-Policy "default-src 'self'; frame-ancestors 'none';" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://127.0.0.1:21117;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 6️⃣ Configuração do Cliente RustDesk

* **ID server:** `SEU_DOMINIO:21116`
* **Relay server:** `SEU_DOMINIO:21117`
* **API server:** `https://SEU_DOMINIO:10447`
* **Key:** `/opt/rustdesk-keys/id_ed25519.pub`

```bash
cat /opt/rustdesk-keys/id_ed25519.pub
```

🖼️ *Sugestão de imagem:* interface do cliente RustDesk mostrando campos de servidor e chave.

---

## 7️⃣ Segurança Adicional

### 🔥 Firewall

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 10447/tcp
sudo ufw allow 21115:21119/tcp
sudo ufw allow 21116/udp
sudo ufw enable
```

### 🛡️ Fail2Ban

```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban --now
```

Exemplo `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
# Tempo que o IP fica banido (em segundos)
bantime = 1d
# Quantas tentativas antes de banir
maxretry = 5
# Tempo para resetar contagem de tentativas
findtime = 10m
# Endereço de email para notificações
destemail = seuemail@dominio.com
sender = fail2ban@localhost
mta = sendmail
banaction = iptables-multiport

# Habilita log detalhado
loglevel = INFO

[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 5

[nginx-http-auth]
enabled = true
port = http,https
filter = nginx-http-auth
logpath = /var/log/nginx/error.log
maxretry = 5

[nginx-botsearch]
enabled = true
port = http,https
filter = nginx-botsearch
logpath = /var/log/nginx/access.log
maxretry = 5

[nginx-limit-req]
enabled = true
port = http,https
filter = nginx-limit-req
logpath = /var/log/nginx/error.log
maxretry = 10

```
