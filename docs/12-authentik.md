# 12 · Authentik (SSO)

Authentik es el **proveedor de identidad (SSO)**: un único inicio de sesión para los servicios compatibles, mediante **OAuth2 / OIDC**.

!!! warning "OAuth2/OIDC, no forward auth"
    En este stack la integración se hace **siempre vía OAuth2/OIDC**. El *forward auth* a través de bloques de configuración personalizados en NPM provoca el error `SSL_ERROR_UNRECOGNIZED_NAME_ALERT` (interferencia del SSL interno de NPM, no es un problema de certificado). Usa OIDC y te evitas el problema.

---

## 1. Despliegue con Docker Compose

Authentik necesita PostgreSQL + Redis. Descarga el compose oficial y crea un `.env`:

```bash
mkdir -p ~/docker/authentik && cd ~/docker/authentik
wget -O docker-compose.yml https://goauthentik.io/docker-compose.yml
nano .env
```

```ini
# .env
PG_PASS=Contraseña_BD_Authentik
AUTHENTIK_SECRET_KEY=Clave_Secreta_Authentik

# Puertos reasignados para evitar conflictos:
#  - 9000 lo ocupa Step-CA
#  - 9443 lo ocupa Portainer
COMPOSE_PORT_HTTP=9200
COMPOSE_PORT_HTTPS=9201
```

!!! tip "Generar los secretos"
    ```bash
    # Clave secreta
    openssl rand -base64 60
    # Contraseña de la base de datos
    openssl rand -base64 36
    ```

```bash
docker compose up -d
```

## 2. Configuración inicial

Accede al asistente de creación del administrador en:

```
http://IP_UBUNTU:9200/if/flow/initial-setup/
```

Define el usuario `akadmin` y su contraseña.

## 3. Triángulo de configuración

1. **DNS:** `auth.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "auth.lan" auth.crt auth.key
   ```
3. **Proxy (Nginx):** Proxy Host `auth.lan` ➔ `http://IP_UBUNTU:9200` + Force SSL.

## 4. Integrar un servicio por OIDC (ejemplo)

1. En Authentik: **Applications → Providers → Create → OAuth2/OpenID Provider**.
2. Anota el **Client ID** y el **Client Secret** que genera.
3. **Redirect URI:** la del servicio (ej. `https://nextcloud.lan/apps/oidc_login/oidc`).
4. Crea la **Application** y enlázala a ese provider.
5. En el servicio destino, configura el cliente OIDC con esos datos y el *issuer* `https://auth.lan/application/o/<slug>/`.

!!! note "Compatibilidad"
    **Sí soportan SSO (OIDC):** Nextcloud, Portainer, BookStack, GLPI, Home Assistant, Immich, n8n.
    **Solo login nativo (sin SSO):** Jellyfin, Vaultwarden, qBittorrent, entre otros.

!!! warning "Excluye su base de datos de Watchtower"
    Añade al servicio `postgresql` del compose:
    ```yaml
        labels:
          - com.centurylinklabs.watchtower.enable=false
    ```
    Un salto de versión mayor de PostgreSQL requiere migrar los datos a mano (ver [18 · Watchtower](18-watchtower.md)).
