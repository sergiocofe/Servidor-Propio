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

---

## 📚 Contenido

| # | Sección |
|---|---------|
| 01 | [Proxmox y máquina virtual](01-proxmox-y-vm.md) |
| 02 | [Docker y Portainer](02-docker-portainer.md) |
| 03 | [Nginx Proxy Manager](03-nginx-proxy-manager.md) |
| 04 | [SSL local con Step-CA](04-ssl-step-ca.md) |
| 05 | [Uptime Kuma](05-uptime-kuma.md) |
| 06 | [Netdata](06-netdata.md) |
| 07 | [Dozzle](07-dozzle.md) |
| 08 | [Homepage](08-homepage.md) |
