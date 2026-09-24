# 19 · Jellyfin (servidor multimedia)

**Jellyfin** es tu servidor de streaming (series, películas, música), 100 % libre, sin cuentas en la nube ni funciones de pago.

---

## 1. Docker Compose

```bash
mkdir -p ~/docker/jellyfin && cd ~/docker/jellyfin
nano docker-compose.yaml
```

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    ports:
      - "8096:8096"
    volumes:
      - /mnt/data/jellyfin/config:/config
      - /mnt/data/jellyfin/cache:/cache
      - /mnt/data/jellyfin/media:/media
    environment:
      - JELLYFIN_PublishedServerUrl=https://jellyfin.lan
```

```bash
docker compose up -d
```

!!! note "Biblioteca compartida"
    `/mnt/data/jellyfin/media` es la biblioteca común: [20 · qBittorrent](20-qbittorrent.md) deposita ahí las descargas e [24 · Immich](24-immich.md) la monta en solo lectura. Así no se duplica espacio.

## Triángulo de configuración

1. **DNS (AdGuard):** `jellyfin.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "jellyfin.lan" jellyfin.crt jellyfin.key
   ```
3. **Proxy (Nginx):** Proxy Host `jellyfin.lan` ➔ `http://IP_UBUNTU:8096` + **Force SSL**.
    - Activa **Websockets Support** (controles remotos y sincronización de reproducción).
4. **Registro:** añádelo a Homepage y a Uptime Kuma.

## Primer acceso

1. Entra en `https://jellyfin.lan`: asistente inicial (idioma, usuario administrador, bibliotecas).
2. Al añadir bibliotecas, apunta a subcarpetas de `/media` (por ejemplo `/media/peliculas`, `/media/series`).
3. Guarda `Usuario_Jellyfin` / `Contraseña_Jellyfin` en Vaultwarden.

!!! info "Login nativo"
    Jellyfin no admite OIDC de forma nativa: mantiene su propio inicio de sesión (ver [12 · Authentik](12-authentik.md)).
