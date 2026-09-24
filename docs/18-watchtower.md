# 18 · Watchtower (actualizaciones automáticas)

Watchtower revisa cada noche si hay versiones nuevas de las imágenes y **recrea los contenedores** con ellas.

!!! danger "Usa el fork mantenido, no `containrrr/watchtower`"
    El proyecto original está archivado: su imagen trae un cliente de la API de Docker antiguo (v1.25) y con un Docker Engine actual entra en bucle de reinicio con:
    ```
    client version 1.25 is too old. Minimum supported API version is 1.40
    ```
    Volver a hacer `pull` no lo arregla (es siempre la misma imagen). Usa el fork activo `ghcr.io/nicholas-fedor/watchtower`.

---

## 1. Docker Compose

```bash
mkdir -p ~/docker/watchtower && cd ~/docker/watchtower
nano docker-compose.yaml
```

```yaml
services:
  watchtower:
    image: ghcr.io/nicholas-fedor/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock   # necesita escribir: socket real, no el proxy
    environment:
      - TZ=Europe/Madrid
      - WATCHTOWER_CLEANUP=true              # borra las imágenes viejas tras actualizar
      - WATCHTOWER_SCHEDULE=0 0 4 * * *      # cada día a las 04:00 (formato con segundos)
```

```bash
docker compose up -d
docker logs watchtower   # debe mostrar la versión y la API negociada, p. ej. "using Docker API v1.5x"
```

## 2. Excluir los contenedores sensibles

Por defecto Watchtower actualiza **todo**. Eso es arriesgado para:

- **Bases de datos** (MariaDB, PostgreSQL, Redis): un salto de versión mayor puede requerir migración manual de los datos.
- **Stacks con versiones sincronizadas**, como Immich: si `immich-server` e `immich-machine-learning` quedan en versiones distintas, pueden dejar de entenderse.

Añade esta etiqueta a cada uno de esos servicios en su `docker-compose.yaml`:

```yaml
    labels:
      - com.centurylinklabs.watchtower.enable=false
```

Y recréalos con `docker compose up -d`. En esta guía la llevan:

| Servicio | Contenedores excluidos |
|---|---|
| [14 · Nextcloud](14-nextcloud.md) | `nextcloud-db` |
| [12 · Authentik](12-authentik.md) | `postgresql` (añádela en su compose) |
| [23 · Paperless-ngx](23-paperless.md) | `paperless-db`, `paperless-redis` |
| [24 · Immich](24-immich.md) | los 4 contenedores (se actualizan a mano, juntos) |

Comprueba qué ha actualizado:

```bash
docker logs watchtower --since 24h
```

!!! tip "Actualizar a mano lo excluido"
    ```bash
    cd ~/docker/immich && docker compose pull && docker compose up -d
    ```
    Lee antes las *release notes* del proyecto y ten una copia reciente ([16 · BorgBackup](16-borgbackup.md)).
