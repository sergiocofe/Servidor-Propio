# 28 · IT-Tools (caja de herramientas)

Colección de utilidades para IT y desarrollo en el navegador: generador de contraseñas, codificadores Base64/JWT, conversores, calculadora de subredes, generador de UUID, hashes...

Todo se ejecuta **en el navegador** (sin backend ni base de datos): los datos que pegas no salen de tu equipo.

---

## 1. Docker Compose

```bash
mkdir -p ~/docker/it-tools && cd ~/docker/it-tools
nano docker-compose.yaml
```

```yaml
services:
  it-tools:
    image: ghcr.io/corentinth/it-tools:latest
    container_name: it-tools
    restart: unless-stopped
    ports:
      - "8087:80"
```

```bash
docker compose up -d
```

!!! note "Sin login"
    No lo necesita: es una web estática sin datos almacenados.

## Triángulo de configuración

1. **DNS (AdGuard):** `tools.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "tools.lan" tools.crt tools.key
   ```
3. **Proxy (Nginx):** Proxy Host `tools.lan` ➔ `http://IP_UBUNTU:8087` + **Force SSL**.
4. **Registro:** añádelo a Homepage y a Uptime Kuma.
