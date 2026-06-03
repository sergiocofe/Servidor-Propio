# 01 · Proxmox y máquina virtual

Preparamos el host Proxmox y creamos la máquina virtual Ubuntu que alojará todos los contenedores Docker.

---

## 1. Acceso a Proxmox (host)

| Parámetro | Valor |
|---|---|
| **URL de acceso** | `https://IP_PROXMOX:8006` |
| **Usuario** | `root` |
| **Contraseña** | `Contraseña_Proxmox` |
| **Nodo** | `casa` |

## 2. Recursos de hardware

- **Disco SSD (sistema):** `local-lvm` / `m22`
- **Disco HDD (datos/multimedia):** `mecanico` (1 TB)
- **Red:** `vmbr0` (rango `192.168.88.x`)

## 3. Especificaciones de la VM (Docker-Master)

| Parámetro | Valor |
|---|---|
| **ID** | `200` |
| **Nombre** | `Ubuntu-Docker` |
| **CPU** | 4 cores |
| **RAM** | 8 GB |
| **Disco** | 80 GB (en almacenamiento `m22`) |

---

## 4. Creación de la VM (paso a paso)

Sigue el asistente **Create VM** en Proxmox con estos parámetros:

| Pestaña | Configuración |
|---|---|
| **General** | ID `200`, nombre `Ubuntu-Docker` |
| **OS** | Seleccionar la ISO de Ubuntu Server |
| **Sistema** | Valores por defecto |
| **Discos** | Almacenamiento `m22` (SSD), tamaño **80 GB** |
| **CPU** | Núcleos: **4** · Tipo: `host` (maximiza instrucciones AES) |
| **Memoria** | **8192 MB** · ⚠️ desmarcar **Ballooning** para memoria dedicada |
| **Red** | Bridge `vmbr0`, modelo `VirtIO` |

---

## 5. Instalación del sistema operativo

Durante la instalación de Ubuntu en la consola:

### 5.1 Red estática

Un servidor necesita IP fija para que los servicios sean accesibles de forma permanente.

| Parámetro | Valor |
|---|---|
| **IPv4 Method** | Manual |
| **Subnet** | `192.168.88.0/24` |
| **Address (IP de la VM)** | `IP_UBUNTU` |
| **Gateway** | `192.168.88.1` |
| **DNS** | `1.1.1.1`, `8.8.8.8` |

### 5.2 Usuario y almacenamiento

- **Usuario:** `Usuario_Ubuntu`
- **Contraseña:** `Contraseña_Usuario`
- **Disco:** marca **Use an entire disk** y **DESMARCA** *Set up this disk as an LVM group*. Simplifica el particionado para Docker.

### 5.3 Servicios esenciales

!!! warning "SSH obligatorio"
    Activa la casilla **[X] Install OpenSSH server**. Permite la administración remota por terminal (PuTTY o VS Code).
