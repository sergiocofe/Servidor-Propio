# 22 · Audiobookshelf (audiolibros y podcasts)

Servidor de audiolibros y podcasts con **seguimiento del progreso de escucha** y apps para móvil.

---

## 1. Docker Compose

```bash
mkdir -p ~/docker/audiobookshelf && cd ~/docker/audiobookshelf
nano docker-compose.yaml
```

```yaml
services:
  audiobookshelf:
    image: ghcr.io/advplyr/audiobookshelf:latest
    container_name: audiobookshelf
    restart: unless-stopped
    ports:
      - "13378:80"
    volumes:
      - /mnt/data/audiobookshelf/audiobooks:/audiobooks
      - /mnt/data/audiobookshelf/podcasts:/podcasts
      - /mnt/data/audiobookshelf/config:/config
      - /mnt/data/audiobookshelf/metadata:/metadata
    environment:
      - TZ=Europe/Madrid
```

```bash
docker compose up -d
```

## Triángulo de configuración

1. **DNS (AdGuard):** `books.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "books.lan" books.crt books.key
   ```
3. **Proxy (Nginx):** Proxy Host `books.lan` ➔ `http://IP_UBUNTU:13378` + **Force SSL**.
    - Activa **Websockets Support** (sincronización del progreso en tiempo real).
4. **Registro:** añádelo a Homepage y a Uptime Kuma.

## Primer acceso

1. Entra en `https://books.lan`: asistente para crear el usuario administrador (`Usuario_Audiobookshelf` / `Contraseña_Audiobookshelf`).
2. Crea las bibliotecas apuntando a `/audiobooks` y `/podcasts`.
3. En la app móvil, usa como servidor `https://books.lan` (con la CA instalada en el móvil, ver [11 · WireGuard](11-wireguard.md)).
