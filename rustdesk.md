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
