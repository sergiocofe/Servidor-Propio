# 08 · Homepage

Un panel de inicio (dashboard) convierte tu laboratorio en una herramienta profesional. **Homepage** es moderno, rápido y se integra con Docker para mostrar estadísticas en tiempo real (contenedores activos, uso de CPU…).

---

## 1. Preparación del entorno

```bash
mkdir -p ~/docker/homepage/config
# Asegurar que tu usuario es el dueño de la carpeta
sudo chown -R Usuario_Ubuntu:Usuario_Ubuntu ~/docker/homepage
cd ~/docker/homepage
nano docker-compose.yaml
```

## 2. Docker Compose

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - 3005:3000
    volumes:
      - ~/docker/homepage/config:/app/config
    environment:
      # Permitimos el acceso desde cualquier host de la red local
      HOMEPAGE_ALLOWED_HOSTS: "*"
    networks:
      - default
      - socket_proxy
    restart: unless-stopped

networks:
  socket_proxy:
    external: true
```

```bash
docker compose up -d
```

!!! warning "Sin acceso directo al socket"
    Homepage **no** monta `/var/run/docker.sock` ni necesita `chmod 666`: lee el estado de los contenedores a través del proxy de solo lectura (ver [02 · Docker y Portainer, apartado 5](02-docker-portainer.md#5-proxy-del-socket-de-docker-solo-lectura)).

---

## 3. Triángulo de configuración (acceso y SSL)

1. **DNS (AdGuard):** *rewrite* → `home.lan` ➔ `IP_UBUNTU`.
2. **Certificado (Step-CA):**
   ```bash
   cd ~/certs && step ca certificate "home.lan" home.crt home.key --not-after=87600h
   ```
3. **Proxy (Nginx):**
    - Sube `home.crt` y `home.key` a NPM.
    - Proxy Host: `home.lan` ➔ `http://IP_UBUNTU:3005`.

---

## 4. Archivos de configuración (`.yaml`)

### A · Conexión a Docker

```bash
nano ~/docker/homepage/config/docker.yaml
```

```yaml
my-docker:
  host: socket-proxy   # nombre del contenedor del proxy (red socket_proxy)
  port: 2375
```

### B · Ajustes visuales

```bash
nano ~/docker/homepage/config/settings.yaml
```

```yaml
title: Sergio Server Dashboard
background: https://images.unsplash.com/photo-1550751827-4bd374c3f58b?q=80&w=1920
```

### C · Widgets de sistema

```bash
nano ~/docker/homepage/config/widgets.yaml
```

```yaml
- system:
    cpu: true
    memory: true
    disk: /
    label: "VM 200 Ubuntu"
- datetime:
    text_size: xl
    format:
      timeStyle: short
      dateStyle: long
```

### D · Tus servicios

```bash
nano ~/docker/homepage/config/services.yaml
```

```yaml
- Infraestructura:
    - Portainer:
        icon: portainer.png
        href: https://portainer.lan
        server: my-docker        # nombre definido en docker.yaml
        container: portainer     # nombre real del contenedor
    - AdGuard Home:
        icon: adguard-home.png
        href: https://adguard.lan
        server: my-docker
        container: adguardhome
        widget:
          type: adguard
          url: http://IP_UBUNTU:3000
          username: Usuario_AdGuard
          password: Contraseña_AdGuard
    - Nginx Proxy Manager:
        icon: nginx-proxy-manager.png
        href: https://nginx.lan
        server: my-docker
        container: nginx-proxy-manager

- Monitorización:
    - Netdata:
        icon: netdata.png
        href: https://netdata.lan
        server: my-docker
        container: netdata
        widget:
          type: netdata
          url: http://IP_UBUNTU:19999
    - Uptime Kuma:
        icon: uptime-kuma.png
        href: https://uptime.lan
        server: my-docker
        container: uptime-kuma
    - Dozzle Logs:
        icon: dozzle.png
        href: https://logs.lan
        server: my-docker
        container: dozzle
```

### Relanzar el panel tras editar la configuración

```bash
cd ~/docker/homepage
docker compose up -d --force-recreate
```
