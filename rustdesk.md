# 🚀 Guia Completo: Servidor RustDesk com Docker, Nginx e SSL

Aprenda a configurar seu próprio servidor **RustDesk** de forma segura, usando Docker, Nginx e Certbot. Ideal para controle total de acesso remoto e autenticação.

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

Exemplo para Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Adicionar chave GPG oficial do Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Adicionar repositório Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

docker --version
docker compose version

# Opcional: permitir Docker sem sudo
sudo usermod -aG docker $USER

sudo mkdir -p /opt/rustdesk-keys

# Rodar container temporário
sudo docker run --name rustdesk-temp -d -p 21115-21119:21115-21119 rustdesk/rustdesk-server hbbs

# Copiar chaves
sudo docker cp rustdesk-temp:/root/id_ed25519 /opt/rustdesk-keys/
sudo docker cp rustdesk-temp:/root/id_ed25519.pub /opt/rustdesk-keys/

# Remover container temporário
sudo docker stop rustdesk-temp
sudo docker rm rustdesk-temp

sudo mkdir -p /opt/rustdesk-server
cd /opt/rustdesk-server
nano docker-compose.yml

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

sudo docker compose up -d
sudo docker compose ps

sudo apt install -y nginx
