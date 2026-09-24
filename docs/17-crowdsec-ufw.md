# 17 · CrowdSec y UFW

Dos capas más de defensa en el host:

- **UFW** es el cortafuegos: decide qué puertos se aceptan.
- **CrowdSec** es un IDS colaborativo: analiza los logs (SSH, Nginx Proxy Manager...), detecta comportamientos maliciosos y **bloquea las IPs** en el cortafuegos mediante un *bouncer*. Además recibe listas de IPs atacantes de la comunidad.

Junto con [10 · Fail2ban](10-fail2ban.md) forman la **seguridad por capas** del servidor.

---

## 1. UFW (cortafuegos)

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22222/tcp          # SSH (puerto personalizado)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 53                 # DNS (AdGuard)
sudo ufw allow from 192.168.88.0/24   # LAN de confianza
sudo ufw allow from 172.16.0.0/12     # tráfico entre contenedores Docker
```

!!! danger "Antes de activar: asegúrate de tener SSH permitido"
    Si activas UFW sin la regla del puerto SSH, te quedas fuera. Comprueba con `sudo ufw show added` antes de `sudo ufw enable`.

### 1.1 UFW + Docker (gotcha)

Docker manipula `iptables` directamente. Sin estos ajustes, el tráfico **entre contenedores se rompe en silencio** (un servicio no alcanza a otro por su IP del host):

```bash
sudo nano /etc/default/ufw
```

```ini
DEFAULT_FORWARD_POLICY="ACCEPT"
```

```bash
sudo nano /etc/ufw/before.rules
```

Añade **al principio del archivo**, antes de la sección `*filter`:

```ini
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 172.16.0.0/12 -j MASQUERADE
COMMIT
```

```bash
sudo ufw enable
sudo ufw reload
sudo ufw status verbose
```

!!! warning "Los puertos publicados por Docker se saltan UFW"
    Un `ports: "8080:80"` en un compose queda accesible aunque UFW no tenga regla para el 8080, porque Docker inserta sus propias reglas antes que las de UFW. Por eso: publica solo los puertos necesarios, mantén el acceso a los servicios siempre a través de NPM y **no abras ningún puerto del servidor en el router**.

---

## 2. CrowdSec (motor de detección)

```bash
curl -s https://install.crowdsec.net | sudo sh
sudo apt install crowdsec -y
```

### 2.1 Cambiar el puerto de la API (conflicto con Vaultwarden)

La API local de CrowdSec usa el `8080` por defecto, que ya ocupa Vaultwarden. Muévela al **8091**:

```bash
sudo nano /etc/crowdsec/config.yaml
```

```yaml
api:
  server:
    listen_uri: 127.0.0.1:8091
```

```bash
sudo nano /etc/crowdsec/local_api_credentials.yaml
```

```yaml
url: http://127.0.0.1:8091
```

```bash
sudo systemctl restart crowdsec
sudo cscli lapi status
```

### 2.2 Colecciones (qué detectar)

```bash
sudo cscli collections install crowdsecurity/linux crowdsecurity/sshd crowdsecurity/nginx-proxy-manager
```

Para que analice los logs de NPM, indícale dónde están:

```bash
sudo nano /etc/crowdsec/acquis.d/npm.yaml
```

```yaml
filenames:
  - /home/Usuario_Ubuntu/docker/nginx-proxy-manager/data/logs/*.log
labels:
  type: nginx-proxy-manager
```

```bash
sudo systemctl restart crowdsec
sudo cscli metrics          # debe mostrar líneas leídas de los logs de SSH y NPM
```

## 3. Bouncer nftables (quien bloquea)

CrowdSec **detecta**; el *bouncer* **bloquea** en el cortafuegos:

```bash
sudo apt install crowdsec-firewall-bouncer-nftables -y
```

Apunta el bouncer a la API en el nuevo puerto:

```bash
sudo nano /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
```

```yaml
api_url: http://127.0.0.1:8091/
```

```bash
sudo systemctl restart crowdsec-firewall-bouncer
sudo cscli bouncers list    # debe aparecer como válido, con un "last pull" reciente
```

## 4. Operaciones útiles

```bash
sudo cscli decisions list                    # IPs bloqueadas ahora mismo
sudo cscli alerts list                       # detecciones recientes
sudo cscli decisions delete --ip DIRECCION_IP   # desbloquear una IP
sudo cscli hub update && sudo cscli hub upgrade # actualizar escenarios
```

!!! tip "No te bloquees a ti mismo"
    Añade tu LAN a la lista blanca:
    ```bash
    sudo cscli parsers install crowdsecurity/whitelists
    ```
    y edita `/etc/crowdsec/parsers/s02-enrich/whitelists.yaml` añadiendo `192.168.88.0/24` en `cidr`.
