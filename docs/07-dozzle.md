# 07 · Dozzle

Dozzle es una herramienta extremadamente ligera para ver los **logs de tus contenedores en tiempo real** sin tocar la terminal.

---

## 1. Despliegue del contenedor

Dozzle lee los logs a través del **proxy del socket de solo lectura** (ver [02 · Docker y Portainer, apartado 5](02-docker-portainer.md#5-proxy-del-socket-de-docker-solo-lectura)), no del socket real.

```bash
mkdir -p ~/docker/dozzle && cd ~/docker/dozzle
nano docker-compose.yaml
```

```yaml
services:
  dozzle:
    container_name: dozzle
    image: amir20/dozzle:latest
    environment:
      - DOCKER_HOST=tcp://socket-proxy:2375
    ports:
      - 8888:8080
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
