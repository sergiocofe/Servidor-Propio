# 23 · Paperless-ngx (gestión documental)

**Paperless-ngx** digitaliza y organiza documentos (facturas, contratos, escaneos) con **OCR** y búsqueda de texto completo.

---

## 1. Secretos en `.env`

```bash
mkdir -p ~/docker/paperless && cd ~/docker/paperless
openssl rand -hex 24   # contraseña de la BD
openssl rand -hex 32   # clave secreta de la aplicación
nano .env
chmod 600 .env
```

```ini
POSTGRES_PASSWORD=Contraseña_BD_Paperless
PAPERLESS_DBPASS=Contraseña_BD_Paperless
PAPERLESS_SECRET_KEY=Clave_Secreta_Paperless
```

!!! danger "`PAPERLESS_SECRET_KEY` es obligatoria"
    Sin ella, Paperless usa una clave por defecto **pública** (la misma para todas las instalaciones) con la que se firman las sesiones. Genérala siempre. Si la cambias en una instalación existente, solo se cierran las sesiones abiertas.

## 2. Docker Compose

```bash
nano docker-compose.yaml
```

```yaml
services:
  paperless-db:
    image: postgres:15
    container_name: paperless-db
    restart: unless-stopped
    env_file:
      - .env
    environment:
      - POSTGRES_DB=paperless
      - POSTGRES_USER=paperless
    volumes:
      - /mnt/data/paperless/db:/var/lib/postgresql/data
    labels:
      - com.centurylinklabs.watchtower.enable=false

  paperless-redis:
    image: redis:7
    container_name: paperless-redis
    restart: unless-stopped
    labels:
      - com.centurylinklabs.watchtower.enable=false

  paperless:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    container_name: paperless
    restart: unless-stopped
    depends_on:
      - paperless-db
      - paperless-redis
    ports:
      - "8083:8000"
    env_file:
      - .env
    environment:
      - PAPERLESS_REDIS=redis://paperless-redis:6379
      - PAPERLESS_DBHOST=paperless-db
      - PAPERLESS_DBNAME=paperless
      - PAPERLESS_DBUSER=paperless
      - PAPERLESS_URL=https://paperless.lan
      - PAPERLESS_TRUSTED_PROXIES=IP_UBUNTU
      - PAPERLESS_OCR_LANGUAGE=spa+eng
      - PAPERLESS_OCR_MODE=skip           # no repite OCR en PDFs que ya tienen texto
      - PAPERLESS_TIME_ZONE=Europe/Madrid
    volumes:
      - /mnt/data/paperless/data:/usr/src/paperless/data
      - /mnt/data/paperless/media:/usr/src/paperless/media
      - /mnt/data/paperless/export:/usr/src/paperless/export
      - /mnt/data/paperless/consume:/usr/src/paperless/consume
```

```bash
docker compose up -d
```

## 3. Crear el administrador

En vez de dejar `PAPERLESS_ADMIN_USER`/`PAPERLESS_ADMIN_PASSWORD` en el compose (solo sirven en el primer arranque), crea el superusuario de forma interactiva:

```bash
docker compose exec paperless python3 manage.py createsuperuser
```

Guarda `Usuario_Paperless` / `Contraseña_Paperless` en Vaultwarden.

## 4. Carpeta `consume` (buzón de entrada)

Todo PDF o imagen que copies en `/mnt/data/paperless/consume` (por red, SMB, desde el escáner...) se procesa con OCR y se archiva automáticamente.

!!! tip "Copia de seguridad nativa"
    Además de Borg, Paperless puede exportar todo (documentos + metadatos) a `/export`:
    ```bash
    docker compose exec paperless document_exporter ../export
    ```

## Triángulo de configuración

1. **DNS (AdGuard):** `paperless.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "paperless.lan" paperless.crt paperless.key
   ```
3. **Proxy (Nginx):** Proxy Host `paperless.lan` ➔ `http://IP_UBUNTU:8083` + **Force SSL**.
4. **Registro:** añádelo a Homepage y a Uptime Kuma.
