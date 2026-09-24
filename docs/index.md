# 🛡️ Servidor propio — Infraestructura blindada

Guía paso a paso para desplegar un servidor de servicios autoalojados, seguro y de **coste de licencias cero**, sobre software libre. El stack prioriza **SSL interno**, **DNS soberano** y **seguridad por capas**, y sirve tanto para uso personal como de referencia profesional.

!!! info "Stack base"
    **SO:** Ubuntu Server · **Orquestación:** Docker + Docker Compose · **Entrada única:** Nginx Proxy Manager · **DNS:** AdGuard Home · **SSL interno:** Step-CA

---

## ⚠️ Sobre las credenciales

Esta guía usa **códigos** (placeholders) en lugar de contraseñas, tokens y claves reales. **Nunca publiques secretos en un repositorio.** Sustituye cada código por tus propios valores en tu copia local.

### Tabla de códigos

| Código | Significado |
|---|---|
| `Usuario_Ubuntu` | Usuario del sistema en la VM |
| `Contraseña_Usuario` | Contraseña de ese usuario |
| `IP_UBUNTU` | IP fija de la VM de Docker |
| `IP_PROXMOX` | IP del host Proxmox |
| `Contraseña_Proxmox` | Contraseña de `root` en Proxmox |
| `Usuario_Nginx` / `Contraseña_Nginx` | Login del panel Nginx Proxy Manager |
| `Usuario_Portainer` / `Contraseña_Portainer` | Credenciales de Portainer |
| `Usuario_AdGuard` / `Contraseña_AdGuard` | Credenciales de AdGuard Home |
| `Contraseña_CA` | Contraseña del certificado raíz de Step-CA |
| `Email_Admin` | Correo del administrador / provisioner |
| `Token_Telegram` | Token del bot de notificaciones |
| `clave_publica_ssh` | Clave pública SSH del cliente |
| `IP_PUBLICA_VPS` | IP pública fija de la VM hub de WireGuard |
| `clave_publica_servidor_wg` / `clave_publica_router_wg` / `clave_publica_cliente_wg` | Claves públicas WireGuard |
| `Contraseña_Root_BD` / `Contraseña_BD_*` | Contraseñas de bases de datos (en `.env`) |
| `Clave_Secreta_Paperless` | `PAPERLESS_SECRET_KEY` |
| `Passphrase_Borg` | Passphrase del repositorio de copias |
| `IP_BACKUP_REMOTO` | Servidor remoto de copias (regla 3-2-1) |
| `Contraseña_MQTT` | Usuario de Mosquitto para Zigbee2MQTT |
| `ID_VM` | ID de la VM en Proxmox |

!!! note
    Las IPs privadas (rango `192.168.88.x`) se dejan como ejemplo concreto para facilitar la lectura. Adáptalas a tu red.

---

## 📦 Requisitos previos

- Un host con **Proxmox VE** (o cualquier hipervisor / máquina física con Ubuntu Server).
- Conocimientos básicos de terminal Linux y SSH.
- Un cliente SSH (PuTTY / OpenSSH) y acceso a la red local.

---

## 🔁 El patrón de 4 pasos

Cada servicio nuevo se despliega con el mismo **"cuadrado de cuatro pasos"**:

1. **Certificado** → genera el SSL con Step-CA.
2. **DNS** → añade el *rewrite* en AdGuard (`servicio.lan` ➔ `IP_UBUNTU`).
3. **Proxy** → crea el *Proxy Host* en Nginx Proxy Manager y activa Force SSL.
4. **Registro** → añade la entrada en Homepage / Uptime Kuma.

Consulta el [mapa de puertos](30-mapa-de-puertos.md) antes de elegir el puerto de un servicio nuevo.

---

## 📚 Contenido

| # | Sección | | # | Sección |
|---|---|---|---|---|
| 01 | [Proxmox y máquina virtual](01-proxmox-y-vm.md) | | 17 | [CrowdSec y UFW](17-crowdsec-ufw.md) |
| 02 | [Docker, Portainer y proxy del socket](02-docker-portainer.md) | | 18 | [Watchtower](18-watchtower.md) |
| 03 | [Nginx Proxy Manager](03-nginx-proxy-manager.md) | | 19 | [Jellyfin](19-jellyfin.md) |
| 04 | [SSL local con Step-CA](04-ssl-step-ca.md) | | 20 | [qBittorrent](20-qbittorrent.md) |
| 05 | [Uptime Kuma](05-uptime-kuma.md) | | 21 | [MeTube](21-metube.md) |
| 06 | [Netdata](06-netdata.md) | | 22 | [Audiobookshelf](22-audiobookshelf.md) |
| 07 | [Dozzle](07-dozzle.md) | | 23 | [Paperless-ngx](23-paperless.md) |
| 08 | [Homepage](08-homepage.md) | | 24 | [Immich](24-immich.md) |
| 09 | [AdGuard Home](09-adguard-home.md) | | 25 | [Stirling PDF](25-stirling-pdf.md) |
| 10 | [Fail2ban](10-fail2ban.md) | | 26 | [Wallos](26-wallos.md) |
| 11 | [WireGuard (VPN con CGNAT)](11-wireguard.md) | | 27 | [Grocy](27-grocy.md) |
| 12 | [Authentik (SSO)](12-authentik.md) | | 28 | [IT-Tools](28-it-tools.md) |
| 13 | [Vaultwarden](13-vaultwarden.md) | | 29 | [Home Assistant + Zigbee2MQTT](29-home-assistant.md) |
| 14 | [Nextcloud](14-nextcloud.md) | | 30 | [Mapa de puertos](30-mapa-de-puertos.md) |
| 15 | [n8n](15-n8n.md) | | 31 | [Mantenimiento e incidencias](31-mantenimiento.md) |
| 16 | [BorgBackup](16-borgbackup.md) | | | |

---

## 🔐 Buenas prácticas de secretos (toda la guía)

- Las contraseñas van en un **`.env`** junto al compose (`chmod 600`), referenciado con `env_file:`, **nunca** escritas en el `docker-compose.yaml`.
- Genera contraseñas con `openssl rand -hex 24`: sin símbolos, así no hay problemas de escape.
- Si un valor lleva `$` (hashes Argon2, por ejemplo), escríbelo como **`$$`** dentro del `.env` (ver [13 · Vaultwarden](13-vaultwarden.md)).
- Añade `.env` a tu `.gitignore`: **nunca** lo subas a un repositorio.
