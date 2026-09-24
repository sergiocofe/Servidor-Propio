# 24 · Immich (fotos y vídeos)

**Immich** es la alternativa autoalojada a Google Fotos: copia automática desde el móvil, reconocimiento facial y búsqueda por contenido (por eso lleva un contenedor de *machine learning* aparte).

---

## 1. Secretos en `.env`

```bash
mkdir -p ~/docker/immich && cd ~/docker/immich
openssl rand -hex 24
nano .env
chmod 600 .env
```

```ini
DB_PASSWORD=Contraseña_BD_Immich
```

## 2. Docker Compose

```bash
nano docker-compose.yaml
```

```yaml
services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:release
    container_name: immich-server
    restart: unless-stopped
    ports:
      - "2283:2283"
    env_file:
      - .env
    environment:
      - DB_HOSTNAME=immich-db
      - DB_USERNAME=immich
      - DB_DATABASE_NAME=immich
      - REDIS_HOSTNAME=immich-redis
    volumes:
      - /mnt/data/immich/upload:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
      - /mnt/data/jellyfin/media:/mnt/data/jellyfin/media:ro   # biblioteca existente, solo lectura
    depends_on:
      - immich-db
      - immich-redis
    labels:
      - com.centurylinklabs.watchtower.enable=false

  immich-machine-learning:
    image: ghcr.io/immich-app/immich-machine-learning:release
    container_name: immich-machine-learning
    restart: unless-stopped
    volumes:
      - /mnt/data/immich/model-cache:/cache
    labels:
      - com.centurylinklabs.watchtower.enable=false

  immich-redis:
    image: redis:7
    container_name: immich-redis
    restart: unless-stopped
    labels:
      - com.centurylinklabs.watchtower.enable=false

  immich-db:
    image: tensorchord/pgvecto-rs:pg14-v0.2.0
    container_name: immich-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=immich
      - POSTGRES_PASSWORD=${DB_PASSWORD}   # se toma del .env
      - POSTGRES_DB=immich
    volumes:
      - /mnt/data/immich/db:/var/lib/postgresql/data
    labels:
      - com.centurylinklabs.watchtower.enable=false
```

```bash
docker compose up -d
```

!!! warning "Imagen de la base de datos"
    Immich necesita un PostgreSQL con extensión vectorial (búsqueda por similitud para caras y objetos): **no** se puede cambiar por un `postgres` genérico. Las versiones recientes de Immich han pasado de `pgvecto-rs` a **VectorChord** (`ghcr.io/immich-app/postgres`). Antes de actualizar, consulta el `docker-compose.yml` oficial de la versión que vayas a instalar y sus notas de migración.

!!! danger "Actualiza los 4 contenedores juntos"
    Todos llevan la etiqueta que los excluye de Watchtower ([18 · Watchtower](18-watchtower.md)). Si `immich-server` e `immich-machine-learning` quedan en versiones distintas pueden dejar de funcionar. Actualízalos a mano, a la vez y tras una copia:
    ```bash
    cd ~/docker/immich && docker compose pull && docker compose up -d
    ```

## Triángulo de configuración

1. **DNS (AdGuard):** `immich.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "immich.lan" immich.crt immich.key
   ```
3. **Proxy (Nginx):** Proxy Host `immich.lan` ➔ `http://IP_UBUNTU:2283` + **Force SSL**.
    - Activa **Websockets Support** (la app móvil lo usa para el progreso de subida).
    - En *Advanced*, añade `client_max_body_size 0;` para permitir subir vídeos grandes.
4. **Registro:** añádelo a Homepage y a Uptime Kuma.

## Primer acceso

1. Entra en `https://immich.lan` y crea el primer usuario administrador (no hay credenciales por defecto).
2. Instala la app móvil, apunta a `https://immich.lan` y activa la **copia de seguridad automática** de la galería.

!!! tip "SSO con Authentik"
    Immich admite OAuth/OIDC (*Administración → Ajustes → Autenticación OAuth*). Ver [12 · Authentik](12-authentik.md).
