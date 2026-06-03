# 02 · Docker y Portainer

Transformamos el servidor base en un nodo de contenedores y le añadimos una interfaz gráfica de gestión.

---

## 1. Actualización del sistema

Antes de instalar software nuevo, sincroniza el repositorio de paquetes:

```bash
sudo apt update && sudo apt upgrade -y
```

Introduce `Contraseña_Usuario` cuando se solicite.

## 2. Instalación de Docker Engine

Usamos el script oficial para garantizar compatibilidad con Ubuntu 24.04:

```bash
# Descargar el script
curl -fsSL https://get.docker.com -o get-docker.sh

# Ejecutar la instalación
sudo sh get-docker.sh

# Limpieza: eliminar el instalador
rm get-docker.sh
```

## 3. Permisos de usuario

Para no usar `sudo` constantemente, añade tu usuario al grupo `docker`:

```bash
sudo usermod -aG docker Usuario_Ubuntu

# Aplicar los cambios sin reiniciar
newgrp docker
```

---

## 4. Despliegue de Portainer CE

Portainer permite administrar contenedores de forma visual desde el navegador.

### 4.1 Volumen de persistencia

```bash
docker volume create portainer_data
```

### 4.2 Lanzamiento del servicio

```bash
docker run -d -p 8000:8000 -p 9443:9443 \
  --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

!!! note "Coherencia con Docker Compose"
    El resto de servicios de esta guía se despliegan con `docker compose`. Portainer es el único que se lanza con `docker run` porque se instala antes que el patrón de carpetas; si prefieres, puedes pasarlo a un `docker-compose.yaml` con los mismos puertos y volúmenes.

### 4.3 Acceso al panel

1. Abre `https://IP_UBUNTU:9443`.
2. **Aviso de seguridad:** clic en *Configuración avanzada* → *Acceder al sitio* (el certificado aún es autofirmado).
3. **Registro inicial:** crea el usuario administrador → `Usuario_Portainer` / `Contraseña_Portainer`.
