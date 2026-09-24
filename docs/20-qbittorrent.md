# 20 · qBittorrent (descargas)

Cliente de torrents con interfaz web, en la imagen de linuxserver.io.

---

## 1. Docker Compose

```bash
mkdir -p ~/docker/qbittorrent && cd ~/docker/qbittorrent
nano docker-compose.yaml
```

```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    ports:
      - "8090:8080"        # interfaz web (el 8080 del host lo ocupa Vaultwarden)
      - "6881:6881"
      - "6881:6881/udp"
    volumes:
      - /mnt/data/qbittorrent/config:/config
      - /mnt/data/qbittorrent/downloads:/downloads
      - /mnt/data/jellyfin/media:/media   # para mover lo descargado a la biblioteca de Jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
      - WEBUI_PORT=8080
```

```bash
docker compose up -d
```

## Triángulo de configuración

1. **DNS (AdGuard):** `torrent.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "torrent.lan" torrent.crt torrent.key
   ```
3. **Proxy (Nginx):** Proxy Host `torrent.lan` ➔ `http://IP_UBUNTU:8090` + **Force SSL**.
4. **Registro:** añádelo a Homepage y a Uptime Kuma.

## Primer acceso

La imagen genera una **contraseña temporal aleatoria** para `admin` en el primer arranque:

```bash
docker logs qbittorrent 2>&1 | grep -i "temporary password"
```

Entra en `https://torrent.lan` con `admin` y esa contraseña, y **cámbiala de inmediato** en *Herramientas → Opciones → Web UI* (`Contraseña_qBittorrent`).

!!! warning "La contraseña temporal cambia en cada reinicio"
    Hasta que no fijes una contraseña propia, cada reinicio genera otra distinta.
