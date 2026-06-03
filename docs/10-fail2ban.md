# 10 · Fail2ban

Fail2ban vigila los registros de acceso y **banea automáticamente** las IPs que fallan repetidamente al autenticarse (ataques de fuerza bruta), sobre todo contra SSH.

!!! note "Por qué no va en Docker"
    A diferencia del resto, Fail2ban se instala en el **host** (no en contenedor): necesita leer los logs del sistema y manipular el cortafuegos directamente para banear IPs.

---

## 1. Instalación

```bash
sudo apt update && sudo apt install fail2ban -y
```

## 2. Configuración (jail.local)

Nunca se edita `jail.conf` (se sobrescribe en actualizaciones). Se crea `jail.local`:

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
# Tiempo de baneo (1 hora) y ventana de detección (10 min)
bantime  = 1h
findtime = 10m
maxretry = 5
# Ignora tu red local para no autobanearte
ignoreip = 127.0.0.1/8 192.168.88.0/24

[sshd]
enabled  = true
port     = 22222         # puerto SSH personalizado (no el 22 por defecto)
filter   = sshd
logpath  = %(sshd_log)s
maxretry = 3
bantime  = 24h
```

## 3. Activar y arrancar

```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
```

## 4. Comprobar el estado

```bash
# Estado general
sudo fail2ban-client status

# Detalle de la cárcel de SSH (IPs baneadas, intentos...)
sudo fail2ban-client status sshd
```

Para desbanear una IP manualmente:

```bash
sudo fail2ban-client set sshd unbanip DIRECCION_IP
```

!!! tip "Defensa en profundidad"
    Fail2ban es la capa reactiva. Combínalo con SSH por clave (puerto `22222`), UFW y CrowdSec para una protección por capas completa.
