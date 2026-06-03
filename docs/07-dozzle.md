# 07 · Dozzle

Dozzle es una herramienta extremadamente ligera para ver los **logs de tus contenedores en tiempo real** sin tocar la terminal.

---

## 1. Despliegue del contenedor

Como Portainer o Netdata, Dozzle se conecta al socket de Docker para leer los logs.

```bash
mkdir -p ~/docker/dozzle && cd ~/docker/dozzle
nano docker-compose.yaml
```

```yaml
services:
  dozzle:
    container_name: dozzle
    image: amir20/dozzle:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - 8888:8080
    restart: unless-stopped
```

```bash
docker compose up -d
```

---

## 2. Triángulo de configuración (acceso y SSL)

1. **DNS (AdGuard):** *rewrite* → `logs.lan` ➔ `IP_UBUNTU`.
2. **Certificado (Ubuntu):**
   ```bash
   cd ~/certs && step ca certificate "logs.lan" logs.crt logs.key
   ```
   (la contraseña del certificado raíz es `Contraseña_CA`).
3. **Proxy (Nginx):**
    - Sube `logs.crt` y `logs.key` a NPM.
    - Proxy Host: `logs.lan` ➔ `http://IP_UBUNTU:8888`.
    - ⚠️ Activa **Websockets Support** (Dozzle lo usa para el streaming de logs en vivo).
