# 31 · Mantenimiento e incidencias

Procedimientos y lecciones aprendidas de incidencias reales del servidor.

---

## 1. El disco de sistema se llena: mover Docker y containerd a `/mnt/data`

Con el uso normal (imágenes, actualizaciones, logs), un disco de sistema pequeño acaba en *"no space left on device"*: se caen bases de datos y fallan las actualizaciones. La solución es mover el almacenamiento de Docker **y** el de containerd al disco de datos.

### 1.1 Docker (`/var/lib/docker`)

```bash
sudo systemctl stop docker docker.socket
sudo rsync -aHAX /var/lib/docker/ /mnt/data/docker/
sudo nano /etc/docker/daemon.json
```

```json
{"data-root": "/mnt/data/docker"}
```

```bash
sudo systemctl start docker
docker info | grep "Docker Root Dir"          # → /mnt/data/docker
sudo du -sh /var/lib/docker /mnt/data/docker  # deben coincidir
sudo mv /var/lib/docker /var/lib/docker.old   # copia de seguridad unos días
```

### 1.2 containerd (`/var/lib/containerd`)

!!! warning "Cambiar el `data-root` de Docker NO mueve containerd"
    containerd es un servicio aparte con su propia configuración. Si no lo mueves, la siguiente actualización vuelve a llenar el disco de sistema.

```bash
sudo systemctl stop docker docker.socket containerd
sudo rsync -aHAX /var/lib/containerd/ /mnt/data/containerd/
sudo nano /etc/containerd/config.toml
```

La línea `root` viene **comentada** por defecto (`#root = "/var/lib/containerd"`). Descoméntala y cambia la ruta:

```toml
root = "/mnt/data/containerd"
```

```bash
sudo systemctl start containerd && sudo systemctl start docker
sudo mv /var/lib/containerd /var/lib/containerd.old
```

Verifica con un `docker compose pull` de cualquier servicio que los datos nuevos se escriben en `/mnt/data/containerd`.

Pasados unos días estable, borra las copias:

```bash
du -sh /var/lib/*.old 2>/dev/null
sudo rm -rf /var/lib/docker.old /var/lib/containerd.old
```

## 2. Tras reiniciar containerd, todos los contenedores quedan "Exited (128)"

containerd pierde la referencia a los procesos en marcha. Los datos están intactos, pero Compose no los recrea solo. Relánzalos todos:

```bash
cd ~/docker
for d in */; do
  echo "=== $d ==="
  (cd "$d" && docker compose up -d)
done
```

No descarga imágenes (están en caché): solo recrea los contenedores.

## 3. ⚠️ Procesos de base de datos duplicados y silenciosos

Tras el paso anterior, pueden quedar **procesos huérfanos** de la tanda anterior (`mariadbd`, `postgres`, `redis-server`) corriendo en paralelo a los nuevos contra **los mismos ficheros de datos**: riesgo real de corrupción.

- MariaDB lo delata: *"Can't lock aria control file ... error: 11"* y bucle de reinicio.
- **PostgreSQL y Redis no dan ningún error visible.** Solo se detectan comparando procesos.

### Detección

```bash
# 1. PID oficial según Docker de cada contenedor con base de datos
for c in $(docker ps --format '{{.Names}}' | grep -Ei 'db|redis|postgres'); do
  echo "$c => $(docker inspect --format '{{.State.Pid}}' "$c")"
done

# 2. Procesos reales en el host
ps -eo pid,lstart,cmd | grep -E "mariadbd|postgres:|redis-server" | grep -v grep
```

Un proceso principal cuya **hora de arranque es muy anterior** al resto y cuyo PID no corresponde a ningún contenedor es un huérfano:

```bash
sudo kill -9 PID_HUERFANO_1 PID_HUERFANO_2
```

Sus procesos hijos (checkpointer, walwriter, conexiones *idle*...) desaparecen solos.

!!! danger "Repite esta comprobación siempre"
    Hazla tras **cualquier reinicio de `containerd` o `dockerd` a nivel de sistema**. Si llegó a haber duplicados durante un tiempo, revisa después la integridad de las aplicaciones afectadas (que no falten archivos o fotos recientes) y ten a mano la última copia de [16 · BorgBackup](16-borgbackup.md).

## 4. Watchtower en bucle de reinicio

`client version 1.25 is too old` → la imagen `containrrr/watchtower` está abandonada. Cambia al fork mantenido (ver [18 · Watchtower](18-watchtower.md)).

## 5. Rutina de revisión

| Frecuencia | Tarea |
|---|---|
| Semanal | `docker logs watchtower --since 7d`: qué se actualizó y si algo falló |
| Semanal | Revisar Uptime Kuma y `sudo cscli alerts list` |
| Mensual | `df -h /` y `df -h /mnt/data`; `docker system df` |
| Mensual | Actualizar a mano lo excluido de Watchtower (bases de datos, Immich), tras una copia |
| Mensual | `borg check` y una restauración de prueba |
| Tras reiniciar Docker/containerd | Diagnóstico de procesos duplicados (apartado 3) |
| Anual | Revisar caducidad de certificados de Step-CA y del certificado raíz |

!!! tip "Limpieza de espacio"
    ```bash
    docker image prune -a      # imágenes sin usar (pide confirmación)
    docker builder prune       # caché de builds
    ```
