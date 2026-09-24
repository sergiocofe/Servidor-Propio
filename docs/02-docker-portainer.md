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

### 4.1 Docker Compose

Como el resto de servicios de la guía, Portainer se despliega con **Docker Compose** (nunca con `docker run`), en su propia carpeta:

```bash
mkdir -p ~/docker/portainer && cd ~/docker/portainer
nano docker-compose.yaml
```

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "8000:8000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock   # Portainer necesita el socket completo (lectura y escritura)
      - portainer_data:/data

volumes:
  portainer_data:
```

```bash
docker compose up -d
```

!!! note "¿Ya lo tenías con `docker run`?"
    Migrarlo no pierde datos si reutilizas el mismo volumen:
    ```bash
    docker stop portainer && docker rm portainer
    ```
    Después, en el compose, declara el volumen como externo para que use el existente:
    ```yaml
    volumes:
      portainer_data:
        external: true
    ```
    y lanza `docker compose up -d`.

### 4.2 Acceso al panel

1. Abre `https://IP_UBUNTU:9443`.
2. **Aviso de seguridad:** clic en *Configuración avanzada* → *Acceder al sitio* (el certificado aún es autofirmado).
3. **Registro inicial:** crea el usuario administrador → `Usuario_Portainer` / `Contraseña_Portainer`.

---

## 5. Proxy del socket de Docker (solo lectura)

Varias herramientas de esta guía (Homepage, Dozzle, Netdata) solo necesitan **leer** el estado de los contenedores. Montarles `/var/run/docker.sock` directamente les da control total sobre Docker, lo que equivale a **root en el host** si alguna de ellas se ve comprometida.

La solución es un **proxy del socket** que solo deja pasar las llamadas de lectura que tú autorices. Los servicios de monitorización hablan con el proxy por una red Docker interna, sin tocar el socket real.

!!! danger "Nunca hagas `chmod 666 /var/run/docker.sock`"
    Hace el socket accesible a **cualquier usuario del sistema**, que podría lanzar un contenedor privilegiado y hacerse root. Si lo aplicaste en algún momento, deshazlo:
    ```bash
    sudo chmod 660 /var/run/docker.sock
    ls -l /var/run/docker.sock   # debe mostrar: srw-rw---- root docker
    ```
    (Reiniciar Docker también restaura los permisos por defecto.)

### 5.1 Red compartida

```bash
docker network create socket_proxy
```

### 5.2 Despliegue

```bash
mkdir -p ~/docker/socket-proxy && cd ~/docker/socket-proxy
nano docker-compose.yaml
```

```yaml
services:
  socket-proxy:
    image: tecnativa/docker-socket-proxy:latest
    container_name: socket-proxy
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      # Solo lectura de lo necesario para dashboards y logs
      - CONTAINERS=1
      - INFO=1
      - IMAGES=1
      - EVENTS=1
      - PING=1
      - VERSION=1
      - POST=0          # bloquea cualquier escritura (crear, parar, borrar...)
    networks:
      - socket_proxy
    # Sin "ports:": el proxy NO se publica en el host, solo es accesible dentro de la red socket_proxy

networks:
  socket_proxy:
    external: true
```

```bash
docker compose up -d
```

A partir de aquí, los servicios que solo leen se conectan a `tcp://socket-proxy:2375` uniéndose a la red `socket_proxy` (ver [06 · Netdata](06-netdata.md), [07 · Dozzle](07-dozzle.md) y [08 · Homepage](08-homepage.md)).

!!! note "Quién sigue usando el socket real"
    **Portainer** y **Watchtower** necesitan escribir (crear, parar, actualizar contenedores), así que mantienen el montaje directo de `/var/run/docker.sock`. Son los únicos.
