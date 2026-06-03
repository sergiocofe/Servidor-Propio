# 13 · Vaultwarden

Vaultwarden es una implementación ligera del servidor de Bitwarden: tu **gestor de contraseñas autoalojado**, compatible con todas las apps y extensiones oficiales de Bitwarden.

!!! note "Login nativo (sin SSO)"
    Vaultwarden usa su propio inicio de sesión; no se integra con Authentik. Es lo correcto para un gestor de contraseñas (su bóveda se cifra con la contraseña maestra del usuario).

---

## 1. Despliegue con Docker Compose

```bash
mkdir -p ~/docker/vaultwarden && cd ~/docker/vaultwarden
nano docker-compose.yaml
```

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    environment:
      DOMAIN: "https://vault.lan"
      ADMIN_TOKEN: "Token_Admin_Vaultwarden"
      SIGNUPS_ALLOWED: "false"      # desactiva el registro abierto tras crear tu cuenta
    volumes:
      - ./data:/data
    ports:
      - "8080:80"
    restart: unless-stopped
```

```bash
docker compose up -d
```

!!! tip "Generar el ADMIN_TOKEN"
    ```bash
    openssl rand -base64 48
    ```
    Ese valor protege el panel de administración en `https://vault.lan/admin`.

## 2. Triángulo de configuración

1. **DNS:** `vault.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "vault.lan" vault.crt vault.key
   ```
3. **Proxy (Nginx):**
    - Proxy Host: `vault.lan` ➔ `http://IP_UBUNTU:8080`.
    - ⚠️ Activa **Websockets Support** (necesario para la sincronización en tiempo real).
    - Marca **Force SSL**.

!!! warning "HTTPS obligatorio"
    El cliente web de Bitwarden/Vaultwarden **exige HTTPS** para funcionar (usa APIs de cifrado del navegador). Por eso es imprescindible el certificado y el Force SSL.
