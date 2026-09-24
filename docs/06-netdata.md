# 06 · Netdata

Netdata muestra qué pasa "bajo el capó" del servidor en **tiempo real**: CPU por contenedor, temperatura, disco… con gráficas por segundo. Complementa a Uptime Kuma (que solo dice si el servicio está vivo).

---

## 1. Despliegue con Docker

```bash
mkdir -p ~/docker/netdata && cd ~/docker/netdata
nano docker-compose.yaml
```

```yaml
services:
  netdata:
    image: netdata/netdata:latest
    container_name: netdata
    hostname: ubuntu-docker
    ports:
      - "19999:19999"
    restart: unless-stopped
    cap_add:
      - SYS_PTRACE
      - NET_ADMIN
    security_opt:
      - apparmor:unconfined
    volumes:
      - netdataconfig:/etc/netdata
      - netdatalib:/var/lib/netdata
      - netdatacache:/var/cache/netdata
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
    environment:
      - DOCKER_HOST=tcp://socket-proxy:2375   # nombres de contenedores vía proxy de solo lectura
    networks:
      - default
      - socket_proxy

volumes:
  netdataconfig:
  netdatalib:
  netdatacache:

networks:
  socket_proxy:
    external: true
```

```bash
docker compose up -d
```

!!! note "Requisito previo"
    Necesitas el proxy del socket desplegado antes (ver [02 · Docker y Portainer, apartado 5](02-docker-portainer.md#5-proxy-del-socket-de-docker-solo-lectura)).

---

## 2. Triángulo de configuración (acceso y SSL)

1. **DNS (AdGuard):** *rewrite* → `netdata.lan` ➔ `IP_UBUNTU`.
2. **Certificado (Ubuntu):**
   ```bash
   cd ~/certs && step ca certificate "netdata.lan" netdata.crt netdata.key
   ```
3. **Proxy (Nginx):**
    - Sube `netdata.crt` y `netdata.key` a NPM.
    - Proxy Host: `netdata.lan` ➔ `http://IP_UBUNTU:19999`.
    - ⚠️ Activa **Websockets Support** (Netdata lo usa para las gráficas en vivo).
