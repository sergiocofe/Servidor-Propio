# 09 · AdGuard Home (DNS)

AdGuard Home es el **DNS soberano** de la red: resuelve los dominios internos `.lan`, bloquea publicidad/rastreadores y es la pieza que usa el "triángulo" de todos los demás servicios.

---

## 1. Despliegue con Docker Compose

```bash
mkdir -p ~/docker/adguardhome && cd ~/docker/adguardhome
nano docker-compose.yaml
```

```yaml
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    network_mode: host          # necesario para servir DNS en el puerto 53
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
    restart: unless-stopped
```

```bash
docker compose up -d
```

!!! warning "Convivencia con Nginx Proxy Manager"
    AdGuard usa `network_mode: host` para ocupar el puerto **53 (DNS)**. Para que **no choque** con NPM en los puertos 80/443, en su configuración interna deja el panel web en el **3000** y desactiva el HTTPS propio (`port_https: 0` en el `AdGuardHome.yaml`); del cifrado ya se encarga NPM.

## 2. Configuración inicial

1. Accede al asistente en `http://IP_UBUNTU:3000`.
2. **Interfaz de administración:** puerto `3000`.
3. **Servidor DNS:** puerto `53`.
4. Crea el usuario admin → `Usuario_AdGuard` / `Contraseña_AdGuard`.

## 3. Reescrituras DNS (lo que da vida a los `.lan`)

En **Filtros → Reescrituras DNS**, añade cada servicio (o un comodín):

| Dominio | Responde a |
|---|---|
| `*.lan` | `IP_UBUNTU` |

Con el comodín `*.lan`, todos los servicios (`portainer.lan`, `nginx.lan`, `vault.lan`…) apuntan a la VM sin tener que darlos de alta uno a uno.

## 4. Triángulo de configuración

1. **DNS:** `adguard.lan` ➔ `IP_UBUNTU` (ya cubierto por el comodín).
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "adguard.lan" adguard.crt adguard.key
   ```
3. **Proxy (Nginx):** Proxy Host `adguard.lan` ➔ `http://IP_UBUNTU:3000` + Force SSL.

!!! tip "Que la red use este DNS"
    Para que toda la red resuelva los `.lan`, configura el **DHCP del router MikroTik** para que entregue `IP_UBUNTU` como servidor DNS, o ponlo manualmente en cada equipo.
