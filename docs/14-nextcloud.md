# 14 · Nextcloud

Nextcloud es tu **nube privada**: archivos, calendario, contactos y mucho más, autoalojado. Aquí lo desplegamos con MariaDB y Redis detrás de Nginx Proxy Manager, con los secretos fuera del compose.

---

## 1. Secretos en `.env`

Las contraseñas **nunca** van escritas en el `docker-compose.yaml`. Genera contraseñas robustas (hexadecimales, así no contienen `$` ni otros símbolos problemáticos):

```bash
openssl rand -hex 24   # repite una vez por cada contraseña
```

```bash
mkdir -p ~/docker/nextcloud && cd ~/docker/nextcloud
nano .env
chmod 600 .env
```

```ini
MYSQL_ROOT_PASSWORD=Contraseña_Root_BD
MYSQL_PASSWORD=Contraseña_BD_Nextcloud
```

!!! warning "Si tu contraseña lleva `$`"
    Compose interpola el `.env`: escribe cada `$` como `$$` (ver [13 · Vaultwarden](13-vaultwarden.md)). Con contraseñas `openssl rand -hex` no hace falta.

## 2. Docker Compose

```bash
nano docker-compose.yaml
```

```yaml
services:
  db:
    image: mariadb:11
    container_name: nextcloud-db
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    env_file:
      - .env
    environment:
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
    volumes:
      - /mnt/data/nextcloud/db:/var/lib/mysql
    labels:
      - com.centurylinklabs.watchtower.enable=false   # la BD no se actualiza sola (ver 18 · Watchtower)
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
      - "8082:80"
    env_file:
      - .env
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      REDIS_HOST: redis
      NEXTCLOUD_TRUSTED_DOMAINS: nextcloud.lan
      TRUSTED_PROXIES: IP_UBUNTU 172.16.0.0/12   # NPM (host y redes Docker)
      OVERWRITEPROTOCOL: https                  # enlaces en https detrás de NPM
    volumes:
      - /mnt/data/nextcloud/data:/var/www/html
    restart: unless-stopped
```

```bash
docker compose up -d
```

!!! note "Datos en el disco grande"
    Base de datos y archivos van a `/mnt/data`, no a una carpeta junto al compose: Nextcloud crece rápido y no debe llenar el disco de sistema.

## 3. Primer acceso

Entra en `https://nextcloud.lan`: el asistente web te pedirá crear el **usuario administrador** (`Usuario_Admin_Nextcloud` / `Contraseña_Admin_Nextcloud`). Los datos de la base de datos ya los toma de las variables.

!!! info "¿Y `NEXTCLOUD_ADMIN_USER` / `NEXTCLOUD_ADMIN_PASSWORD`?"
    Esas variables **solo se leen en el primer arranque con la base de datos vacía**. Con Nextcloud ya instalado no hacen nada, así que no las dejes en el compose: solo conservarían una contraseña vieja en texto plano.

Para cambiar después la contraseña de un usuario, sin reiniciar nada:

```bash
docker exec -u www-data -e OC_PASS='Nueva_Contraseña' nextcloud \
  php occ user:resetpassword --password-from-env Usuario_Admin_Nextcloud
```

Comprueba que las variables de proxy se aplicaron:

```bash
docker exec -u www-data nextcloud php occ config:system:get overwriteprotocol   # → https
docker exec -u www-data nextcloud php occ config:system:get trusted_proxies
```

## 4. Triángulo de configuración

1. **DNS:** `nextcloud.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "nextcloud.lan" nextcloud.crt nextcloud.key
   ```
3. **Proxy (Nginx):** Proxy Host `nextcloud.lan` ➔ `http://IP_UBUNTU:8082` + Force SSL.

!!! warning "Dominios de confianza y proxy"
    Si ves *"Acceso a través de un dominio no fiable"*, revisa `NEXTCLOUD_TRUSTED_DOMAINS` (o `trusted_domains` en `config/config.php`). `OVERWRITEPROTOCOL` evita el contenido mixto y `TRUSTED_PROXIES` hace que Nextcloud vea la IP real del cliente.

## 5. Cambiar contraseñas de la BD en un Nextcloud que ya funciona

!!! danger "Gotchas reales"
    - **La contraseña del `.env` debe coincidir con la que ya está activa dentro de MariaDB.** Si solo editas el archivo y recreas, la app no podrá conectar (caída). Para mover una contraseña existente al `.env`, extráela del contenedor en vez de reescribirla a mano:
      ```bash
      docker inspect nextcloud-db --format '{{range .Config.Env}}{{println .}}{{end}}' | grep MYSQL
      ```
    - **`docker compose up -d app` también recrea `db`** si la configuración de la BD cambió en el archivo: Compose recrea las dependencias de `depends_on` aunque no las nombres.
    - Para rotarla de verdad: primero `ALTER USER` dentro de MariaDB, después actualiza el `.env` y recrea.

!!! tip "SSO opcional con Authentik"
    Nextcloud admite OIDC. Instala la app *OpenID Connect Login* y configúrala contra Authentik (ver [12 · Authentik](12-authentik.md)).
