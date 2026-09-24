# 21 · MeTube (descarga de vídeos)

Interfaz web sencilla sobre `yt-dlp` para descargar vídeos de YouTube y cientos de sitios más.

---

## 1. Docker Compose

```bash
mkdir -p ~/docker/metube && cd ~/docker/metube
nano docker-compose.yaml
```

```yaml
services:
  metube:
    image: ghcr.io/alexta69/metube:latest
    container_name: metube
    restart: unless-stopped
    ports:
      - "8081:8081"
    volumes:
      - /mnt/data/metube/downloads:/downloads
    environment:
      - TZ=Europe/Madrid
```

```bash
docker compose up -d
```

!!! warning "Sin inicio de sesión"
    MeTube **no tiene login propio**: cualquiera que llegue a `https://metube.lan` puede usarlo. Mantenlo accesible solo desde la LAN y la VPN ([11 · WireGuard](11-wireguard.md)), nunca publicado a internet.

## Triángulo de configuración

1. **DNS (AdGuard):** `metube.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "metube.lan" metube.crt metube.key
   ```
3. **Proxy (Nginx):** Proxy Host `metube.lan` ➔ `http://IP_UBUNTU:8081` + **Force SSL**.
4. **Registro:** añádelo a Homepage y a Uptime Kuma.
