# 03 · Nginx Proxy Manager

Desplegamos el **proxy inverso**, el punto de entrada único que redirige el tráfico web y centraliza los certificados SSL.

---

## 1. Organización del espacio de trabajo

Cada aplicación tiene su propio directorio de configuración:

```bash
cd ~
mkdir -p docker/nginx-proxy-manager
cd docker/nginx-proxy-manager
```

## 2. Despliegue con Docker Compose

Crea el archivo de configuración:

```bash
nano docker-compose.yaml
```

Pega el siguiente contenido:

```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'    # Tráfico web estándar
      - '81:81'    # Panel de administración
      - '443:443'  # Tráfico web seguro (HTTPS)
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

Guarda (`Ctrl+O`, `Enter`) y sal (`Ctrl+X`).

## 3. Puesta en marcha

```bash
docker compose up -d
docker ps          # verifica que el contenedor está corriendo
```

---

## 4. Configuración inicial del administrador

1. Accede vía web al puerto **81**: `http://IP_UBUNTU:81`.
2. Credenciales temporales por defecto: `admin@example.com` / `changeme`.
3. El sistema te obligará a cambiarlas → usa `Usuario_Nginx` / `Contraseña_Nginx`.
