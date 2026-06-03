# 11 · WireGuard (VPN)

WireGuard es una VPN moderna, rápida y sencilla. Te permite acceder de forma segura a tu red `.lan` desde fuera. Aquí montamos el **servidor con wg-easy** (gestión visual de clientes) y configuramos clientes en **Linux, Windows y MikroTik**.

---

## 1. Servidor: wg-easy (Docker)

`wg-easy` levanta WireGuard y añade un panel web para crear y descargar configuraciones de cliente con un clic.

```bash
mkdir -p ~/docker/wg-easy && cd ~/docker/wg-easy
nano docker-compose.yaml
```

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:latest
    container_name: wg-easy
    environment:
      - WG_HOST=IP_PUBLICA_VPS        # IP o dominio público desde donde te conectarás
      - PASSWORD_HASH=Hash_Contraseña_WG  # hash de la contraseña del panel
      - WG_DEFAULT_DNS=IP_UBUNTU      # para resolver los .lan dentro del túnel
      - WG_DEFAULT_ADDRESS=10.8.0.x
    volumes:
      - ./config:/etc/wireguard
    ports:
      - "51820:51820/udp"            # tráfico VPN
      - "51821:51821/tcp"            # panel web
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

```bash
docker compose up -d
```

Accede al panel en `http://IP_UBUNTU:51821`, crea un cliente y descarga su `.conf` o escanea el QR.

!!! warning "Rendimiento sobre VPS (MTU)"
    Si publicas WireGuard a través de un VPS (Oracle Cloud free tier, etc.), el túnel es sensible al MTU. Si notas que carga lento o se cuelga: baja el **MTU a 1360** en la interfaz, aplica **MSS clamping** y excluye el tráfico VPN de FastTrack en el router.

---

## 2. Cliente Linux

```bash
sudo apt install wireguard -y
```

Copia la configuración descargada a `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = clave_privada_wg
Address = 10.8.0.2/24
DNS = IP_UBUNTU
MTU = 1360

[Peer]
PublicKey = clave_publica_servidor_wg
PresharedKey = preshared_key_wg
Endpoint = IP_PUBLICA_VPS:51820
AllowedIPs = 192.168.88.0/24, 10.8.0.0/24
PersistentKeepalive = 25
```

Levantar / bajar el túnel:

```bash
sudo wg-quick up wg0
sudo wg-quick down wg0
# Arranque automático:
sudo systemctl enable wg-quick@wg0
```

---

## 3. Cliente Windows

1. Descarga la app oficial desde **https://www.wireguard.com/install/**.
2. **Add Tunnel → Import tunnel(s) from file** y selecciona el `.conf` que descargaste de wg-easy (o **Add empty tunnel** y pega el contenido del bloque de arriba).
3. Pulsa **Activate**.

!!! tip
    Marca *Block untunneled traffic (kill-switch)* solo si quieres que **todo** el tráfico vaya por la VPN. Para acceder solo a la `.lan`, deja `AllowedIPs = 192.168.88.0/24, 10.8.0.0/24`.

---

## 4. Cliente / Peer en MikroTik (RouterOS)

Útil para que **toda** la red de una sede salga por la VPN, o como peer permanente.

```bash
# 1. Crear la interfaz WireGuard
/interface/wireguard add name=wg-vpn listen-port=51820 mtu=1360

# 2. Ver la clave pública generada (la necesitarás en el servidor)
/interface/wireguard print

# 3. Asignar IP a la interfaz
/ip/address add address=10.8.0.3/24 interface=wg-vpn

# 4. Añadir el peer (el servidor)
/interface/wireguard/peers add interface=wg-vpn \
  public-key="clave_publica_servidor_wg" \
  preshared-key="preshared_key_wg" \
  endpoint-address=IP_PUBLICA_VPS endpoint-port=51820 \
  allowed-address=192.168.88.0/24,10.8.0.0/24 \
  persistent-keepalive=25s

# 5. MSS clamping (evita problemas de MTU sobre el túnel)
/ip/firewall/mangle add chain=forward protocol=tcp tcp-flags=syn \
  action=change-mss new-mss=clamp-to-pmtu out-interface=wg-vpn
```

!!! note
    Da de alta el MikroTik como un cliente más en el panel de wg-easy para obtener su `PresharedKey` y la `clave_publica_servidor_wg`, y registra en el servidor la clave pública que te muestra el `print` del paso 2.
