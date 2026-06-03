# 14 · Nextcloud

Nextcloud es tu **nube privada**: archivos, calendario, contactos y mucho más, autoalojado. Aquí lo desplegamos con MariaDB y Redis detrás de Nginx Proxy Manager.

---

## 1. Despliegue con Docker Compose

```bash
mkdir -p ~/docker/nextcloud && cd ~/docker/nextcloud
nano docker-compose.yaml
```

```yaml
services:
  db:
    image: mariadb:11
    container_name: nextcloud-db
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    environment:
      MYSQL_ROOT_PASSWORD: Contraseña_Root_BD
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: Contraseña_BD_Nextcloud
    volumes:
      - ./db:/var/lib/mysql
    restart: unless-stopped

  redis:
    image: redis:alpine
    container_name: nextcloud-redis
    restart: unless-stopped

  app:
    image: nextcloud:latest
    container_name: nextcloud
    depends_on:
      - db
      - redis
    ports:
      - "8081:80"
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: Contraseña_BD_Nextcloud
      REDIS_HOST: redis
      NEXTCLOUD_ADMIN_USER: Usuario_Admin_Nextcloud
      NEXTCLOUD_ADMIN_PASSWORD: Contraseña_Admin_Nextcloud
      NEXTCLOUD_TRUSTED_DOMAINS: nextcloud.lan
      OVERWRITEPROTOCOL: https        # imprescindible detrás de NPM
    volumes:
      - ./html:/var/www/html
    restart: unless-stopped
```

```bash
docker compose up -d
```

## 2. Triángulo de configuración

1. **DNS:** `nextcloud.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "nextcloud.lan" nextcloud.crt nextcloud.key
   ```
3. **Proxy (Nginx):** Proxy Host `nextcloud.lan` ➔ `http://IP_UBUNTU:8081` + Force SSL.

!!! warning "Dominios de confianza y proxy"
    Si ves *"Acceso a través de un dominio de confianza no fiable"*, añade el dominio en `config/config.php` (`trusted_domains`) o vía variable `NEXTCLOUD_TRUSTED_DOMAINS`. La variable `OVERWRITEPROTOCOL: https` evita los avisos de contenido mixto al ir detrás del proxy.

!!! tip "SSO opcional con Authentik"
    Nextcloud admite OIDC. Instala la app *OpenID Connect Login* y configúrala contra Authentik (ver [12 · Authentik](12-authentik.md)) para usar el inicio de sesión único.
