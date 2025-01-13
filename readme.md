# Pterodactyl Panel and Node Setup Guide

This guide walks through setting up both the Pterodactyl panel and node components using Docker Compose.

## Panel Setup

### 1. Create Panel Directory
```bash
mkdir pterodactyl/panel/
cd pterodactyl/panel/
```

### 2. Create Docker Compose Configuration
Create `docker-compose.yml` with the following configuration:

```yaml
version: '3.8'
x-common:
  database:
    &db-environment
    MYSQL_PASSWORD: &db-password  "CHANGE_ME"
    MYSQL_ROOT_PASSWORD: "CHANGE_ME_TOO"
  panel:
    &panel-environment
    APP_URL: "https://pterodactyl.example.com"
    APP_TIMEZONE: "UTC"
    APP_SERVICE_AUTHOR: "noreply@example.com"
    TRUSTES_PROXIES: "*"
  mail:
    &mail-environment
    MAIL_FROM: "noreply@example.com"
    MAIL_DRIVER: "smtp"
    MAIL_HOST: "mail"
    MAIL_PORT: "1025"
    MAIL_USERNAME: ""
    MAIL_PASSWORD: ""
    MAIL_ENCRYPTION: "true"
services:
  database:
    image: mariadb:10.5
    restart: always
    command: --default-authentication-plugin=mysql_native_password
    volumes:
      - db:/var/lib/mysql
    environment:
      <<: *db-environment
      MYSQL_DATABASE: "panel"
      MYSQL_USER: "pterodactyl"
  cache:
    image: redis:alpine
    restart: always
  panel:
    image: ghcr.io/pterodactyl/panel:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    links:
      - database
      - cache
    volumes:
      - panel_var:/app/var 
      - panel_nginx:/etc/nginx/http.d/
      - panel_certs:/etc/letsencrypt/
      - panel_logs:/app/storage/logs/
volumes:
  db:
  panel_var:
  panel_nginx:
  panel_certs:
  panel_logs:

    environment:
      <<: [*panel-environment, *mail-environment]
      DB_HOST: "database"
      APP_ENV: "production"
      APP_ENVIRONMENT_ONLY: "false"
      CACHE_DRIVER: "redis"
      SESSION_DRIVER: "redis"
      QUEUE_DRIVER: "redis"
      REDIS_HOST: "cache"
      DB_PORT: "3306"
      DB_PASSWORD: *db-password

networks:
  default:
    ipam:
      config:
        - subnet: "172.20.0.0/16"
```

### 3. Start Panel Services
```bash
docker-compose up -d
```

### 4. Create Admin User
```bash
docker-compose run --rm panel php artisan p:user:make
```

**Important**: Make sure port 8080 is publicly accessible.

## Node (Wings) Setup

### 1. Create Wings Directory
```bash
mkdir pterodactyl/wings/
cd pterodactyl/wings/
```

### 2. Create Wings Docker Compose Configuration
Create `docker-compose.yml` with the following configuration:

```yaml
version: '3.8'

services:
  wings:
    image: ghcr.io/pterodactyl/wings:v1.6.1
    restart: always
    network:
      - wings0
    ports:
      - "8080:8080"
      - "2022:2022"
      - "443:443"
    tty: true
    environment:
      TZ: "UTC"
      WINGS_UID: 988
      WINGS_GID: 988
      WINGS_USERNAME: pterodactyl
    volumes:
  - docker_sock:/var/run/docker.sock
  - docker_containers:/var/lib/docker/containers/
  - pterodactyl_config:/etc/pterodactyl/
  - pterodactyl_data:/var/lib/pterodactyl/
  - pterodactyl_logs:/var/log/pterodactyl/
  - pterodactyl_tmp:/tmp/pterodactyl/
  - ssl_certs:/etc/ssl/certs:ro

volumes:
  docker_sock:
  docker_containers:
  pterodactyl_config:
  pterodactyl_data:
  pterodactyl_logs:
  pterodactyl_tmp:
  ssl_certs:

networks:
  wings0:
    name: wings0
    driver: bridge
    ipam:
      config:
        - subnet: "127.21.0.0/16"
      driver_opts:
        com.docker.network.bridge.name: wings0
```

### 3. Configure and Start Wings
```bash
cd pterodactyl
sudo su
nano /etc/pterodactyl/config.yml  # Configure your wings settings here
cd wings
docker-compose up -d --force-recreate
```

## Important Notes

1. Remember to modify the following in the panel configuration:
   - Database passwords (`CHANGE_ME` and `CHANGE_ME_TOO`)
   - App URL
   - Email settings
   - Trusted proxies

2. For the wings setup:
   - Ensure the config.yml is properly configured with your panel's details
   - Verify all required ports are accessible (8080, 2022, 443)
   - Make sure the Docker socket is properly mounted

3. Security considerations:
   - Always change default passwords
   - Use proper SSL/TLS certificates
   - Configure firewalls appropriately
   - Keep both panel and wings updated regularly
