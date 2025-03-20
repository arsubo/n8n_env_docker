# Configuración de n8n con PostgreSQL, NGINX y Let's Encrypt

Este documento detalla los pasos para configurar **n8n** con **PostgreSQL**, utilizando **NGINX** como **Reverse Proxy** y **Let's Encrypt** para certificados SSL en un servidor basado en Docker Compose.

---

## 1️⃣ **Preparar el servidor**

### 1.1 **Actualizar el sistema**
```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 **Instalar dependencias**
```bash
sudo apt install -y docker.io docker-compose certbot python3-certbot-nginx
```

### 1.3 **Configurar el Firewall**
```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

---

## 2️⃣ **Configurar el entorno de trabajo**

### 2.1 **Crear directorio de trabajo**
```bash
mkdir -p ~/n8n-compose && cd ~/n8n-compose
```

### 2.2 **Crear el archivo `.env` con las variables de entorno**
```bash
touch .env
nano .env
```

Ejemplo de contenido: ver archivo tmp.env y cambiar valores renombrando archivo como .env
```ini
N8N_PORT=5678
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_DATABASE=n8n_db
DB_POSTGRESDB_USER=n8n_user
DB_POSTGRESDB_PASSWORD=super_secure_password
```

---

## 3️⃣ **Configurar Docker Compose**

### 3.1 **Crear `docker-compose.yml`**
```bash
touch docker-compose.yml
nano docker-compose.yml
```

Ejemplo de contenido:
```yaml
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

---

## 4️⃣ **Configurar NGINX**

### 4.1 **Crear archivo de configuración `nginx.conf`**
```bash
touch nginx.conf
nano nginx.conf
```

Ejemplo de configuración:
```nginx
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

  # Aumentar el tamaño máximo de archivos permitidos en n8n
    client_max_body_size 128M;

server {
    listen 443 ssl http2;
    server_name n8n.example.com;

    ssl_certificate /etc/letsencrypt/live/n8n.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.example.com/privkey.pem;
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

---

## 5️⃣ **Obtener certificados SSL con Let's Encrypt**

### 5.1 **Detener NGINX si está corriendo**
```bash
docker stop n8n-compose-nginx-1
```

### 5.2 **Generar los certificados SSL**
```bash
sudo certbot certonly --standalone -d n8n.example.com
```

### 5.3 **Verificar la renovación automática**
```bash
sudo certbot renew --dry-run
```

---

## 6️⃣ **Levantar los servicios**
```bash
docker compose up -d
```

Verifica que los contenedores estén corriendo:
```bash
docker ps
```

---

## 7️⃣ **Verificar NGINX y n8n**

### 7.1 **Verificar logs de NGINX**
```bash
docker logs -f n8n-compose-nginx-1
```

### 7.2 **Verificar acceso a n8n desde NGINX**
```bash
docker exec -it n8n-compose-nginx-1 curl -I http://n8n:5678
```

✅ **Si todo está bien, accede a:**
```plaintext
https://n8n.example.com
```

---

## 🎯 **Conclusión**
Este setup te permite correr **n8n** con **PostgreSQL** detrás de un **NGINX Reverse Proxy** con **SSL/TLS** habilitado mediante **Let's Encrypt**. 🚀

Si hay problemas, revisa los logs:
```bash
docker logs -f n8n-compose-nginx-1