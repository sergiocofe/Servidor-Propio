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
      SIGNUPS_ALLOWED: "false"      # desactiva el registro abierto tras crear tu cuenta
    env_file:
      - .env                        # aquí vive el ADMIN_TOKEN (nunca en el compose)
    volumes:
      - ./data:/data
    ports:
      - "8080:80"
    restart: unless-stopped
```

```bash
docker compose up -d
```

### ADMIN_TOKEN (panel `/admin`) con hash Argon2

El panel de administración (`https://vault.lan/admin`) se protege con `ADMIN_TOKEN`. Si lo dejas vacío, el panel queda **desactivado** (comportamiento seguro). Para activarlo, usa un **hash Argon2**, no un token en texto plano:

```bash
docker run --rm -it vaultwarden/server /vaultwarden hash
```

Te pide la contraseña de acceso a `/admin` y devuelve algo como `$argon2id$v=19$m=65540,t=3,p=4$...$...`.

Guárdalo en el `.env`:

```bash
nano ~/docker/vaultwarden/.env
```

```ini
ADMIN_TOKEN=$$argon2id$$v=19$$m=65540,t=3,p=4$$Hash_Sal_Vaultwarden$$Hash_Valor_Vaultwarden
```

!!! danger "Escapa cada `$` como `$$`"
    Docker Compose **interpola también el `.env`**: cada `$` se interpretaría como una variable (`$v`, `$m`...) y el hash llegaría roto. Síntomas: avisos `WARN ... variable is not set` al hacer `docker compose up` y el log de Vaultwarden diciendo *"you are using a plain text ADMIN_TOKEN"*. La solución es duplicar **todos** los `$` del hash.

Aplica y verifica:

```bash
docker compose up -d --force-recreate
docker logs vaultwarden --tail 10   # sin WARN ni "plain text" = correcto
```

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
