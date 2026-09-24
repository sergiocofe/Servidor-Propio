# 27 · Grocy (inventario doméstico)

Gestión de la casa: despensa, **fechas de caducidad**, lista de la compra, recetas y tareas del hogar.

---

## 1. Docker Compose

```bash
mkdir -p ~/docker/grocy && cd ~/docker/grocy
nano docker-compose.yaml
```

```yaml
services:
  grocy:
    image: lscr.io/linuxserver/grocy:latest
    container_name: grocy
    restart: unless-stopped
    ports:
      - "8086:80"
    volumes:
      - /mnt/data/grocy:/config
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
```

```bash
docker compose up -d
```

## Triángulo de configuración

1. **DNS (AdGuard):** `grocy.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "grocy.lan" grocy.crt grocy.key
   ```
3. **Proxy (Nginx):** Proxy Host `grocy.lan` ➔ `http://IP_UBUNTU:8086` + **Force SSL**.
4. **Registro:** añádelo a Homepage y a Uptime Kuma.

## Primer acceso

!!! danger "Credenciales por defecto: `admin` / `admin`"
    Entra en `https://grocy.lan` y **cámbialas en el primer inicio de sesión** (*Gestión de usuarios → admin → Cambiar contraseña*) por `Contraseña_Grocy`.

Configura idioma, moneda y unidades en *Ajustes* antes de empezar a cargar productos.
