# 16 · BorgBackup

BorgBackup hace **copias de seguridad deduplicadas y cifradas**. Es muy eficiente: solo guarda los bloques que cambian, así que las copias diarias ocupan muy poco.

!!! note "Se instala en el host"
    Como Fail2ban, Borg va en el **host** (no en contenedor): necesita acceso al sistema de archivos para respaldar volúmenes Docker, configuraciones y datos de usuario.

---

## 1. Instalación

```bash
sudo apt update && sudo apt install borgbackup -y
```

## 2. Inicializar el repositorio (cifrado)

Por ejemplo, en el disco mecánico de datos:

```bash
borg init --encryption=repokey-blake2 /mnt/mecanico/borg-repo
```

!!! danger "Guarda la passphrase y la clave"
    Borg te pedirá una contraseña (`Passphrase_Borg`). **Sin ella no podrás restaurar nada.** Guárdala en tu gestor de contraseñas (Vaultwarden) y exporta también la clave del repo:
    ```bash
    borg key export /mnt/mecanico/borg-repo ~/borg-key-backup.txt
    ```

## 3. Script de copia de seguridad

```bash
nano ~/borg-backup.sh
```

```bash
#!/bin/bash
export BORG_REPO="/mnt/mecanico/borg-repo"
export BORG_PASSPHRASE="Passphrase_Borg"

# Crea una copia con fecha en el nombre
borg create --stats --compression zstd \
  ::'{hostname}-{now:%Y-%m-%d}' \
  /home/Usuario_Ubuntu/docker \
  /etc \
  --exclude '*/node_modules' \
  --exclude '*.tmp'

# Política de retención: 7 diarias, 4 semanales, 6 mensuales
borg prune --list \
  --keep-daily=7 --keep-weekly=4 --keep-monthly=6

borg compact
```

Dale permisos de ejecución y protégelo (contiene la passphrase):

```bash
chmod 700 ~/borg-backup.sh
```

## 4. Programar la copia diaria (cron)

```bash
crontab -e
```

Añade (copia diaria a las 03:00):

```cron
0 3 * * * /home/Usuario_Ubuntu/borg-backup.sh >> /home/Usuario_Ubuntu/borg.log 2>&1
```

## 5. Operaciones útiles

```bash
# Listar las copias del repositorio
borg list /mnt/mecanico/borg-repo

# Ver el contenido de una copia concreta
borg list /mnt/mecanico/borg-repo::NOMBRE_COPIA

# Restaurar (extrae en el directorio actual)
borg extract /mnt/mecanico/borg-repo::NOMBRE_COPIA
```

!!! tip "Copia 3-2-1 y borgmatic"
    Para cumplir la regla 3-2-1, replica el repo a un destino externo (otro disco o servidor remoto por SSH). Si quieres una gestión más cómoda de programación y retención, **borgmatic** envuelve a Borg con un único archivo de configuración YAML.
