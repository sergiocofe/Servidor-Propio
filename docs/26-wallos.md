# 26 · Wallos (gestión de suscripciones)

Control de **suscripciones recurrentes** (streaming, software, dominios...): coste mensual y anual, fechas de renovación y avisos antes de que te cobren.

---

## 1. Docker Compose

```bash
mkdir -p ~/docker/wallos && cd ~/docker/wallos
nano docker-compose.yaml
```

```yaml
services:
  wallos:
    image: bellamy/wallos:latest
    container_name: wallos
    restart: unless-stopped
    ports:
      - "8085:80"
    volumes:
      - /mnt/data/wallos:/var/www/html/db
    environment:
      - TZ=Europe/Madrid
```

```bash
docker compose up -d
```

## Triángulo de configuración

1. **DNS (AdGuard):** `wallos.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "wallos.lan" wallos.crt wallos.key
   ```
3. **Proxy (Nginx):** Proxy Host `wallos.lan` ➔ `http://IP_UBUNTU:8085` + **Force SSL**.
4. **Registro:** añádelo a Homepage y a Uptime Kuma.

## Primer acceso

Entra en `https://wallos.lan`: la aplicación pide crear el usuario administrador (`Usuario_Wallos` / `Contraseña_Wallos`). Después, en ajustes, desactiva el registro de nuevos usuarios si no lo vas a compartir.

!!! tip "Avisos de renovación"
    Wallos puede notificar por Telegram o correo antes de cada renovación: reutiliza el `Token_Telegram` del bot del servidor.
