# 16 · BorgBackup

BorgBackup hace **copias de seguridad deduplicadas y cifradas**: solo guarda los bloques que cambian, así que las copias diarias ocupan muy poco.

!!! note "Se instala en el host"
    Como Fail2ban, Borg va en el **host** (no en contenedor) y se ejecuta como **root**: necesita leer todos los datos de `/mnt/data`, que pertenecen a usuarios distintos según el contenedor.

---

## 0. Qué hay que respaldar (y qué no)

| Ruta | ¿Copia? | Motivo |
|---|---|---|
| `~/docker` | ✅ | Todos los `docker-compose.yaml`, `.env` y configuraciones |
| `/etc` | ✅ | Configuración del sistema (UFW, Fail2ban, containerd...) |
| `~/.step` | ✅ | **Tu CA.** Si la pierdes, tendrás que reemitir y reinstalar todos los certificados |
| `/mnt/data` | ✅ | **Aquí viven los datos de verdad**: Nextcloud, Immich, Paperless, Jellyfin... |
| Carpetas de BD en caliente (`/mnt/data/*/db`) | ❌ | Copiar ficheros de una BD en marcha da copias inconsistentes. Se respaldan con **volcados** (apartado 3) |
| `/mnt/data/docker`, `/mnt/data/containerd` | ❌ | Imágenes y capas: se vuelven a descargar |
| Cachés (`model-cache`, `jellyfin/cache`) | ❌ | Regenerables |

!!! danger "El repositorio no puede estar dentro de lo que copias"
    Usa un **disco distinto** para el repo (en el ejemplo, `/mnt/backup`). Una copia en el mismo disco no te protege de que ese disco muera.

## 1. Instalación

```bash
sudo apt update && sudo apt install borgbackup -y
```

## 2. Passphrase fuera del script e inicialización del repo

La passphrase va en un archivo que solo root puede leer, nunca escrita dentro del script:

```bash
sudo sh -c 'openssl rand -hex 32 > /root/.borg-passphrase'
sudo chmod 600 /root/.borg-passphrase
sudo cat /root/.borg-passphrase   # = Passphrase_Borg → guárdala en Vaultwarden AHORA
```

```bash
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" \
  borg init --encryption=repokey-blake2 /mnt/backup/borg-repo
```

!!! danger "Guarda la passphrase y la clave"
    **Sin ellas no podrás restaurar nada.** Guarda `Passphrase_Borg` en tu gestor de contraseñas y exporta la clave del repo fuera del servidor:
    ```bash
    sudo borg key export /mnt/backup/borg-repo /root/borg-key-backup.txt
    ```

## 3. Script de copia con volcados de bases de datos

```bash
sudo nano /usr/local/sbin/borg-backup.sh
```

```bash
#!/bin/bash
set -euo pipefail

export BORG_REPO="/mnt/backup/borg-repo"
export BORG_PASSCOMMAND="cat /root/.borg-passphrase"
DUMPS="/mnt/data/_dumps"
mkdir -p "$DUMPS"

# 1) Volcados consistentes de las bases de datos (en caliente, sin parar servicios)
docker exec nextcloud-db sh -c \
  'mariadb-dump --single-transaction -uroot -p"$MYSQL_ROOT_PASSWORD" --all-databases' \
  > "$DUMPS/nextcloud.sql"

for c in immich-db paperless-db authentik-postgresql-1; do
  docker exec "$c" sh -c 'pg_dumpall -U "$POSTGRES_USER"' > "$DUMPS/$c.sql"
done

# 2) Copia deduplicada y cifrada
borg create --stats --compression zstd \
  ::'{hostname}-{now:%Y-%m-%d}' \
  /home/Usuario_Ubuntu/docker \
  /home/Usuario_Ubuntu/.step \
  /etc \
  /mnt/data \
  --exclude '/mnt/data/docker' \
  --exclude '/mnt/data/containerd' \
  --exclude '/mnt/data/*/db' \
  --exclude '/mnt/data/immich/model-cache' \
  --exclude '/mnt/data/jellyfin/cache' \
  --exclude '*/node_modules' \
  --exclude '*.tmp'

# 3) Retención: 7 diarias, 4 semanales, 6 mensuales
borg prune --list --keep-daily=7 --keep-weekly=4 --keep-monthly=6
borg compact
```

```bash
sudo chmod 700 /usr/local/sbin/borg-backup.sh
```

!!! note "Adapta los nombres"
    Comprueba los nombres reales de tus contenedores de BD con `docker ps --format '{{.Names}}'`. Si añades servicios con base de datos, añade su volcado al script.

!!! tip "Biblioteca multimedia"
    Si `/mnt/data/jellyfin/media` es muy grande y recuperable (descargas), puedes excluirla también para ahorrar espacio. Las fotos de Immich (`/mnt/data/immich/upload`) **no** son recuperables: esas sí, siempre.

## 4. Programar la copia diaria (cron de root)

```bash
sudo crontab -e
```

```cron
0 3 * * * /usr/local/sbin/borg-backup.sh >> /var/log/borg-backup.log 2>&1
```

!!! note "Hora de la copia"
    A las 03:00, antes de que Watchtower actualice a las 04:00 (ver [18 · Watchtower](18-watchtower.md)): si una actualización rompe algo, tienes la copia de justo antes.

## 5. Copia externa (regla 3-2-1)

3 copias, en 2 soportes distintos, **1 fuera de casa**. Borg replica a un servidor remoto por SSH (otro servidor tuyo, el de un familiar o un servicio compatible con Borg):

```bash
# Inicializar el repo remoto (una vez)
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" \
  borg init --encryption=repokey-blake2 ssh://usuario_remoto@IP_BACKUP_REMOTO:22/./borg-repo
```

Añade al final del script un segundo `borg create` apuntando a ese repo (con `BORG_REPO` distinto), o automatízalo con **borgmatic**, que gestiona varios repositorios, retención y volcados de BD desde un único YAML.

## 6. Operaciones útiles

```bash
export BORG_PASSCOMMAND="cat /root/.borg-passphrase"

sudo -E borg list /mnt/backup/borg-repo                      # listar copias
sudo -E borg list /mnt/backup/borg-repo::NOMBRE_COPIA        # contenido de una copia
sudo -E borg extract /mnt/backup/borg-repo::NOMBRE_COPIA     # restaurar (en el directorio actual)
sudo -E borg check /mnt/backup/borg-repo                     # verificar integridad
```

!!! warning "Una copia no probada no es una copia"
    Haz de vez en cuando una restauración de prueba: extrae un volcado `.sql` y un par de archivos en una carpeta temporal y comprueba que están bien.
