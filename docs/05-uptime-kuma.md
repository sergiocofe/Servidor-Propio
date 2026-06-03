# 05 · Uptime Kuma

Panel para supervisar que todos tus servicios `.lan` estén funcionando, con alertas y tiempo de actividad.

---

## 5.1 Despliegue del contenedor

### Paso 1 · Preparar el entorno

```bash
mkdir -p ~/docker/uptime-kuma && cd ~/docker/uptime-kuma
nano docker-compose.yaml
```

### Paso 2 · Docker Compose

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    volumes:
      - ./data:/app/data
    ports:
      - "3001:3001"
    restart: unless-stopped
```

### Paso 3 · Levantar el servicio

```bash
docker compose up -d
```

---

## 5.2 Acceso local seguro (HTTPS)

### Paso 1 · Generar el certificado

```bash
cd ~/certs
step ca certificate "uptime.lan" uptime.crt uptime.key
```

### Paso 2 · Descargar los archivos (desde tu PC, PowerShell)

```powershell
cd $HOME\Desktop\Certificados_Guia
scp Usuario_Ubuntu@IP_UBUNTU:~/certs/uptime.crt .
scp Usuario_Ubuntu@IP_UBUNTU:~/certs/uptime.key .
```

---

## 5.3 Integración en Nginx Proxy Manager

1. **Añadir certificado:** *SSL Certificates → Add Custom Certificate*. Sube `uptime.key`, `uptime.crt` y usa `root_ca.crt` como intermedio.
2. **Crear Proxy Host:**
    - **Domain Name:** `uptime.lan`
    - **Scheme:** `http` (Uptime Kuma interno usa HTTP)
    - **Forward Port:** `3001`
    - **Websockets Support:** **ACTIVO** (obligatorio para las gráficas)
    - **SSL:** selecciona el certificado y marca **Force SSL**

---

## 5.4 Configurar monitores de red

Al entrar por primera vez en `https://uptime.lan` crearás tu usuario administrador (`Usuario_Ubuntu` / `Contraseña_Usuario`). Luego añade un monitor:

1. **Añadir nuevo monitor**.
2. **Tipo:** HTTP(s).
3. **URL:** `https://portainer.lan`.
4. Si da error de certificado local, marca **Ignorar errores de TLS/SSL**.
