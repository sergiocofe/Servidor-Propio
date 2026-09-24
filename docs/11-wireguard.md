# 11 · WireGuard (VPN)

Objetivo: conectarte desde fuera de casa (móvil, portátil) y acceder a toda la red `192.168.88.0/24` como si estuvieras dentro (`https://portainer.lan`, `https://n8n.lan`...), **sin exponer nada de tu casa a internet** y sin depender de servicios de terceros (Tailscale, Cloudflare...).

---

## 0. Por qué un hub externo: el problema del CGNAT

Muchos operadores ponen al cliente detrás de **CGNAT** (Carrier-Grade NAT): tu IP pública la comparten otros clientes, así que **ningún reenvío de puertos en tu router funcionará nunca**. No te llega tráfico entrante, hagas lo que hagas.

La solución es invertir la dirección: una pequeña **VM con IP pública fija** (por ejemplo, la capa gratuita *Always Free* de Oracle Cloud o cualquier VPS barato) hace de **punto de encuentro (hub)**. Tu router de casa y tus dispositivos se conectan **hacia fuera** a ese hub (las conexiones salientes sí funcionan con CGNAT) y el hub los pone en contacto.

```
 Móvil ──┐                          ┌── Router casa (MikroTik) ── LAN 192.168.88.0/24
         └──►  VM hub (IP pública) ◄┘
               10.66.66.1
```

!!! tip "¿Cómo sé si tengo CGNAT?"
    Compara la IP WAN que muestra tu router con la de `https://ifconfig.me`. Si no coinciden, o la del router está en `100.64.0.0/10`, estás detrás de CGNAT.

### Direccionamiento de la VPN

| Elemento | IP en el túnel |
|---|---|
| Hub (VM pública) | `10.66.66.1/24` |
| Router de casa | `10.66.66.2/24` |
| Cliente 1 (móvil) | `10.66.66.10/32` |
| Cliente 2 | `10.66.66.11/32` (y así sucesivamente) |

---

## 1. Crear la VM hub (ejemplo: Oracle Cloud Always Free)

1. **Compute → Instances → Create Instance.** Elige una región cercana.
2. **Imagen:** cámbiala explícitamente a **Canonical Ubuntu** (por defecto propone Oracle Linux; esta guía usa `apt`).
3. **Shape:** `VM.Standard.E2.1.Micro` (1 OCPU / 1 GB) es **suficiente** para un hub WireGuard con pocos dispositivos.
4. **Red:** si es tu primera instancia, *Create new virtual cloud network* + *Create new public subnet*.
5. **SSH:** *Generate a key pair for me* y descarga la clave privada.

!!! warning "Gotcha: la IP pública puede no asignarse sola"
    Si la instancia arranca sin IP pública: **Instancia → Networking → VNIC primaria → IP administration → ⋮ → Edit → Public IP type: Reserved public IP → Create new**. Así queda una IP **fija** (`IP_PUBLICA_VPS`) que no cambia al parar/arrancar la VM, y no necesitas DuckDNS.

## 2. Abrir el puerto en el firewall de la nube (Security List)

La nube bloquea el tráfico **antes** de llegar al sistema operativo. En la subred de la instancia → **Security List** por defecto → **Add Ingress Rules**:

- **Source CIDR:** `0.0.0.0/0`
- **IP Protocol:** UDP
- **Destination Port:** `51820`

## 3. Preparar la VM

```bash
chmod 600 ruta/a/clave.key
ssh -i ruta/a/clave.key ubuntu@IP_PUBLICA_VPS
sudo apt update && sudo apt upgrade -y
sudo apt install -y wireguard
```

### 3.1 Claves del servidor

```bash
umask 077
wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
sudo chmod 600 /etc/wireguard/privatekey
sudo cat /etc/wireguard/publickey   # = clave_publica_servidor_wg (anótala)
```

!!! note "No hagas `cd /etc/wireguard`"
    El directorio es `700` (solo root): el `cd` como usuario normal da *Permission denied*. Por eso se usan rutas absolutas.

### 3.2 Configuración del servidor (primero SIN peers)

```bash
sudo tee /etc/wireguard/wg0.conf > /dev/null <<EOT
[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = $(sudo cat /etc/wireguard/privatekey)
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT
EOT
sudo chmod 600 /etc/wireguard/wg0.conf
```

!!! warning "Arranca sin bloques `[Peer]`"
    Un `[Peer]` con un placeholder en vez de una clave real hace fallar `wg-quick` (*Key is not the correct length or format*). Añade cada peer cuando tengas su clave real.

## 4. Reenvío de paquetes y firewall local

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Las imágenes Ubuntu de Oracle traen reglas `iptables` de fábrica con un `REJECT` que bloquea todo salvo SSH. Hay que corregir **dos cadenas**:

**INPUT** — inserta la regla de WireGuard **antes** del `REJECT` (sustituye `X` por el número de línea del `REJECT`):

```bash
sudo iptables -L INPUT -n --line-numbers
sudo iptables -I INPUT X -p udp --dport 51820 -j ACCEPT
```

**FORWARD** — aquí el `REJECT` viene **en la primera posición**, así que las `ACCEPT` que añade el `PostUp` nunca se ejecutan. Síntoma: el túnel hace *handshake* pero no llega nada a la LAN (`ERR_ADDRESS_UNREACHABLE`). Muévelo al final:

```bash
sudo iptables -L FORWARD -n --line-numbers -v
sudo iptables -D FORWARD -j REJECT --reject-with icmp-host-prohibited
sudo iptables -A FORWARD -j REJECT --reject-with icmp-host-prohibited
```

Guarda las reglas y arranca WireGuard:

```bash
sudo netfilter-persistent save      # si no existe: sudo apt install -y iptables-persistent
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

---

## 5. Router de casa (MikroTik) como peer saliente

El router **no abre ningún puerto**: solo se conecta hacia el hub.

```bash
# 1. Interfaz y su IP en el túnel
/interface wireguard add name=wireguard-hub comment="VPN via hub (CGNAT)"
/ip address add address=10.66.66.2/24 interface=wireguard-hub

# 2. Clave pública del router (para el servidor)
/interface wireguard print

# 3. El hub como peer
/interface wireguard peers add interface=wireguard-hub \
  public-key="clave_publica_servidor_wg" \
  endpoint-address=IP_PUBLICA_VPS endpoint-port=51820 \
  allowed-address=10.66.66.0/24 \
  persistent-keepalive=25s comment="Hub VPN"

# 4. Permitir que el tráfico del túnel llegue a la LAN
/ip firewall filter add chain=forward in-interface=wireguard-hub action=accept \
  comment="VPN -> LAN" place-before=0
```

!!! warning "`persistent-keepalive` es obligatorio"
    Detrás de CGNAT, el hueco de NAT hacia el hub se cierra si no hay tráfico. El keepalive cada 25 s lo mantiene abierto.

En el **hub**, registra el router. Su `AllowedIPs` incluye la LAN de casa: así el hub sabe que `192.168.88.0/24` está detrás del router.

```bash
sudo tee -a /etc/wireguard/wg0.conf > /dev/null <<EOT

# Peer: router de casa
[Peer]
PublicKey = clave_publica_router_wg
AllowedIPs = 10.66.66.2/32, 192.168.88.0/24
EOT
sudo systemctl restart wg-quick@wg0
```

Prueba desde el router: `/ping 10.66.66.1`.

---

## 6. Clientes (móvil, portátil)

1. Instala la app oficial **WireGuard** (en Android también está en F-Droid) y crea un **túnel vacío**: la app genera sus claves. Copia su clave pública.
2. En el hub, añade el peer y reinicia:
   ```bash
   sudo tee -a /etc/wireguard/wg0.conf > /dev/null <<EOT

   # Peer: Cliente 1
   [Peer]
   PublicKey = clave_publica_cliente_wg
   AllowedIPs = 10.66.66.10/32
   EOT
   sudo systemctl restart wg-quick@wg0
   ```
3. En la app:
    - **Address:** `10.66.66.10/32`
    - **DNS:** `IP_UBUNTU` (AdGuard, imprescindible para resolver los `.lan`)
    - **Endpoint:** `IP_PUBLICA_VPS:51820`
    - **Public key (peer):** `clave_publica_servidor_wg`
    - **Allowed IPs:** `192.168.88.0/24, 10.66.66.0/24` (*split tunnel*: solo va por la VPN el tráfico hacia casa)
    - **Persistent keepalive:** `25`

Equivalente en un **cliente Linux** (`/etc/wireguard/wg0.conf`, luego `sudo wg-quick up wg0`):

```ini
[Interface]
PrivateKey = clave_privada_cliente_wg
Address = 10.66.66.11/32
DNS = IP_UBUNTU

[Peer]
PublicKey = clave_publica_servidor_wg
Endpoint = IP_PUBLICA_VPS:51820
AllowedIPs = 192.168.88.0/24, 10.66.66.0/24
PersistentKeepalive = 25
```

En **Windows**, la app oficial permite *Add empty tunnel* y pegar ese mismo bloque.

### Confiar en tu CA desde el móvil

Para que `https://*.lan` no dé aviso, instala el certificado raíz de Step-CA. Sírvelo temporalmente desde el servidor de casa:

```bash
cd ~/.step/certs && python3 -m http.server 8099   # Ctrl+C al terminar
```

Con la VPN activa, descarga `http://IP_UBUNTU:8099/root_ca.crt` en el móvil e instálalo (Android: *Ajustes → Seguridad → Cifrado y credenciales → Instalar certificado → Certificado de CA*).

---

## 7. Diagnóstico

Prueba siempre con **datos móviles**, no con el WiFi de casa. Si falla, revisa en este orden:

1. **No resuelve `.lan`:** el DNS del túnel debe ser `IP_UBUNTU`.
2. **No hay handshake:** Security List (apartado 2) y cadena INPUT (apartado 4). `sudo wg show` debe mostrar un *latest handshake* reciente.
3. **Handshake sí, pero no llega a la LAN:** cadena FORWARD del hub (apartado 4), `AllowedIPs` del router en el hub con `192.168.88.0/24`, y regla forward del MikroTik.
4. **Carga lenta o se cuelga en páginas grandes (MTU):** baja el MTU a `1360` en los extremos y aplica MSS clamping en el MikroTik:
   ```bash
   /ip firewall mangle add chain=forward protocol=tcp tcp-flags=syn \
     action=change-mss new-mss=clamp-to-pmtu out-interface=wireguard-hub
   ```

---

## 8. Hardening del hub

Es la única pieza con puertos abiertos a internet (22/TCP y 51820/UDP). WireGuard es silencioso (descarta sin responder lo no autenticado), pero SSH hay que reforzarlo:

```bash
# 1. Confirmar que SSH no acepta contraseñas (debe salir "PasswordAuthentication no")
sudo grep -Ei '^(PasswordAuthentication|PermitRootLogin)' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf

# 2. Fail2ban para SSH
sudo apt install -y fail2ban
sudo tee /etc/fail2ban/jail.local > /dev/null <<EOT
[sshd]
enabled = true
port = ssh
backend = systemd
maxretry = 5
findtime = 10m
bantime = 1h
EOT
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd   # si da "Failed to access socket", repite en 2 s

# 3. Parches de seguridad automáticos
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

!!! tip "Portabilidad"
    Toda la VPN es un archivo `wg0.conf`. Si el proveedor cambia sus condiciones gratuitas, migrar a cualquier VPS es copiar ese archivo y repetir los apartados 2 a 4.
