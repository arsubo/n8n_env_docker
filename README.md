n8n Deployment Guide | Guía de Despliegue de n8n
================================================

Choose your language / Selecciona tu idioma:

-   [English version](https://www.google.com/search?q=%23english "null")

-   [Versión en Español](https://www.google.com/search?q=%23espa%C3%B1ol "null")

<a name="english"></a>

🇺🇸 n8n Setup with PostgreSQL, NGINX and Let's Encrypt
=======================================================

This document details the steps to configure **n8n** with **PostgreSQL**, using **NGINX** as a **Reverse Proxy** and **Let's Encrypt** for SSL certificates on a Docker Compose-based server.

1️⃣ Prepare the Server
----------------------

### 1.1 Update the system

```
sudo apt update && sudo apt upgrade -y

```

### 1.2 Install dependencies

```
sudo apt install -y docker.io docker-compose certbot python3-certbot-nginx

```

### 1.3 Configure the Firewall

```
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

```

2️⃣ Configure the Working Environment
-------------------------------------

### 2.1 Create working directory

```
mkdir -p ~/n8n-compose && cd ~/n8n-compose

```

### 2.2 Create the `.env` file with environment variables

```
touch .env
nano .env

```

Example content (rename your file from `tmp.env` to `.env` and change values):

```
N8N_PORT=5678
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_DATABASE=n8n_db
DB_POSTGRESDB_USER=n8n_user
DB_POSTGRESDB_PASSWORD=super_secure_password

```

3️⃣ Configure Docker Compose
----------------------------

### 3.1 Create `docker-compose.yml`

```
version: '3.8'

services:
  postgres:
    image: postgres:13
    restart: always
    environment:
      POSTGRES_DB: ${DB_POSTGRESDB_DATABASE}
      POSTGRES_USER: ${DB_POSTGRESDB_USER}
      POSTGRES_PASSWORD: ${DB_POSTGRESDB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - n8n-network

  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=${DB_POSTGRESDB_DATABASE}
      - DB_POSTGRESDB_USER=${DB_POSTGRESDB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=adminpassword
    depends_on:
      - postgres
    networks:
      - n8n-network

  nginx:
    image: nginx:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - n8n
    networks:
      - n8n-network

volumes:
  postgres_data:

networks:
  n8n-network:
    driver: bridge

```

4️⃣ Configure NGINX
-------------------

### 4.1 Create the `nginx.conf` configuration file

```
touch nginx.conf
nano nginx.conf

```

Configuration example (replace `n8n.example.com` with your domain):

```
server {
    listen 80;
    server_name n8n.example.com;
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}

client_max_body_size 128M;

server {
    listen 443 ssl http2;
    server_name n8n.example.com;

    ssl_certificate /etc/letsencrypt/live/[n8n.example.com/fullchain.pem](https://n8n.example.com/fullchain.pem);
    ssl_certificate_key /etc/letsencrypt/live/[n8n.example.com/privkey.pem](https://n8n.example.com/privkey.pem);
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://n8n:5678;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

5️⃣ Obtain SSL Certificates with Let's Encrypt
----------------------------------------------

### 5.1 Stop NGINX if it's running

```
docker stop n8n-compose-nginx-1

```

### 5.2 Generate SSL certificates

```
sudo certbot certonly --standalone -d n8n.example.com

```

6️⃣ Start the Services
----------------------

```
docker compose up -d

```

✅ **If everything is correct, access at:** `https://n8n.example.com`

[Back to top / Volver arriba](https://www.google.com/search?q=%23english "null")

<a name="español"></a>

🇲🇽 Configuración de n8n con PostgreSQL, NGINX y Let's Encrypt
===============================================================

Este documento detalla los pasos para configurar **n8n** con **PostgreSQL**, utilizando **NGINX** como **Reverse Proxy** y **Let's Encrypt** para certificados SSL en un servidor basado en Docker Compose.

1️⃣ Preparar el servidor
------------------------

### 1.1 Actualizar el sistema

```
sudo apt update && sudo apt upgrade -y

```

### 1.2 Instalar dependencias

```
sudo apt install -y docker.io docker-compose certbot python3-certbot-nginx

```

2️⃣ Configurar el entorno de trabajo
------------------------------------

### 2.1 Crear directorio de trabajo

```
mkdir -p ~/n8n-compose && cd ~/n8n-compose

```

### 2.2 Crear el archivo `.env`

```
touch .env && nano .env

```

Ejemplo de contenido:

```
N8N_PORT=5678
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_DATABASE=n8n_db
DB_POSTGRESDB_USER=n8n_user
DB_POSTGRESDB_PASSWORD=super_secure_password

```

3️⃣ Configurar Docker Compose
-----------------------------

*(Usa el archivo YAML detallado en la sección superior).*

4️⃣ Configurar NGINX
--------------------

*(Usa el archivo `nginx.conf` detallado en la sección superior, reemplazando tu dominio real).*

5️⃣ Obtener certificados SSL
----------------------------

```
sudo certbot certonly --standalone -d n8n.example.com

```

6️⃣ Levantar los servicios
--------------------------

```
docker compose up -d

```

✅ **Acceso:** `https://n8n.example.com`

🎯 Conclusión
-------------

Este setup permite correr **n8n** de forma segura y profesional con persistencia de datos en PostgreSQL.

[Volver arriba](https://www.google.com/search?q=%23english "null")
