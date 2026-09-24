# 25 · Stirling PDF (herramientas de PDF)

Caja de herramientas para PDFs autoalojada: unir, dividir, convertir, comprimir, firmar, OCR... **Los documentos nunca salen de tu servidor**, a diferencia de las webs gratuitas de "convertir PDF online".

---

## 1. Docker Compose

```bash
mkdir -p ~/docker/stirling-pdf && cd ~/docker/stirling-pdf
nano docker-compose.yaml
```

```yaml
services:
  stirling-pdf:
    image: frooodle/s-pdf:latest
    container_name: stirling-pdf
    restart: unless-stopped
    ports:
      - "8084:8080"
    volumes:
      - /mnt/data/stirling-pdf/configs:/configs
      - /mnt/data/stirling-pdf/logs:/logs
    environment:
      - DOCKER_ENABLE_SECURITY=false   # sin login (ver aviso)
      - LANGS=es_ES
```

```bash
docker compose up -d
```

!!! warning "Sin autenticación"
    Con `DOCKER_ENABLE_SECURITY=false` la herramienta **no pide login**: cualquiera en la LAN o la VPN puede usarla. Si vas a tratar documentación sensible, activa el inicio de sesión propio de Stirling (variable `true` y usuario inicial). Los nombres de estas variables han cambiado entre versiones: consulta la documentación de la versión instalada.

!!! note "Nombre de la imagen"
    El proyecto publica ahora su imagen como `stirlingtools/stirling-pdf`; `frooodle/s-pdf` es el nombre histórico. Si al actualizar deja de recibir versiones, cambia el `image:`.

## Triángulo de configuración

1. **DNS (AdGuard):** `pdf.lan` ➔ `IP_UBUNTU`.
2. **Certificado:**
   ```bash
   cd ~/certs && step ca certificate "pdf.lan" pdf.crt pdf.key
   ```
3. **Proxy (Nginx):** Proxy Host `pdf.lan` ➔ `http://IP_UBUNTU:8084` + **Force SSL**.
4. **Registro:** añádelo a Homepage y a Uptime Kuma.
