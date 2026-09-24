# 29 · Home Assistant + Zigbee2MQTT + Mosquitto

Domótica local, sin nube de fabricantes: **Home Assistant** es el cerebro, **Zigbee2MQTT** habla con los dispositivos Zigbee (persianas, sensores, enchufes) a través de un dongle USB, y **Mosquitto** es el bus MQTT que los comunica.

```
Dispositivos Zigbee ─► Dongle USB ─► Zigbee2MQTT ─► Mosquitto (MQTT) ─► Home Assistant
```

!!! info "¿Por qué Zigbee2MQTT y no ZHA?"
    Zigbee2MQTT tiene mejor soporte para dispositivos genéricos Tuya (muy habituales en módulos de persiana baratos) y expone más opciones de cada dispositivo.

---

## 1. El dongle USB

Ejemplo con **Sonoff Zigbee 3.0 USB Dongle Plus V2** (chip CH9102, ID USB `1a86:55d4`).

Si el servidor es una VM de Proxmox, pasa el USB a la VM desde el host:

```bash
qm set ID_VM -usb0 host=1a86:55d4
```

Dentro de la VM, localiza su ruta estable (no uses `/dev/ttyUSB0`, que puede cambiar al reiniciar):

```bash
ls -l /dev/serial/by-id/
# usb-ITEAD_SONOFF_Zigbee_3.0_USB_Dongle_Plus_V2_XXXXXXXX-if00 -> ../../ttyUSB0
```

!!! tip "Alcance: aleja el dongle del servidor"
    Los puertos USB 3.0 generan interferencias en 2,4 GHz. Conecta el dongle con un **alargador USB** de 1 m, lejos de la torre. Es la mejora de alcance más barata.

## 2. Mosquitto (preparación)

```bash
sudo mkdir -p /mnt/data/mosquitto/{config,data,log}
sudo nano /mnt/data/mosquitto/config/mosquitto.conf
```

```ini
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd

persistence true
persistence_location /mosquitto/data/

log_dest file /mosquitto/log/mosquitto.log
```

!!! danger "Gotcha: crea `passwd` antes de arrancar"
    Si el `password_file` no existe, Mosquitto entra en bucle de reinicio:
    ```bash
    sudo touch /mnt/data/mosquitto/config/passwd
    sudo chown -R 1883:1883 /mnt/data/mosquitto
    ```

## 3. Secretos en `.env`

```bash
mkdir -p ~/docker/home-assistant && cd ~/docker/home-assistant
openssl rand -hex 24
nano .env
chmod 600 .env
```

```ini
ZIGBEE_USB_ID=usb-ITEAD_SONOFF_Zigbee_3.0_USB_Dongle_Plus_V2_XXXXXXXX-if00
MQTT_USER=z2m
MQTT_PASSWORD=Contraseña_MQTT
```

## 4. Docker Compose

```bash
nano docker-compose.yaml
```

```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    volumes:
      - /mnt/data/mosquitto/config:/mosquitto/config
      - /mnt/data/mosquitto/data:/mosquitto/data
      - /mnt/data/mosquitto/log:/mosquitto/log
    networks:
      - iot

  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:latest
    container_name: zigbee2mqtt
    restart: unless-stopped
    depends_on:
      - mosquitto
    ports:
      - "8088:8080"
    volumes:
      - /mnt/data/zigbee2mqtt/data:/app/data
    devices:
      - /dev/serial/by-id/${ZIGBEE_USB_ID}:/dev/serial/by-id/${ZIGBEE_USB_ID}
    environment:
      - TZ=Europe/Madrid
      - ZIGBEE2MQTT_CONFIG_SERIAL_PORT=/dev/serial/by-id/${ZIGBEE_USB_ID}
      - ZIGBEE2MQTT_CONFIG_MQTT_SERVER=mqtt://mosquitto:1883
      - ZIGBEE2MQTT_CONFIG_MQTT_USER=${MQTT_USER}
      - ZIGBEE2MQTT_CONFIG_MQTT_PASSWORD=${MQTT_PASSWORD}
      - ZIGBEE2MQTT_CONFIG_FRONTEND_PORT=8080
      - ZIGBEE2MQTT_CONFIG_PERMIT_JOIN=false
      # ⚠️ NO añadas ZIGBEE2MQTT_CONFIG_HOMEASSISTANT=true (ver gotchas)
    networks:
      - iot

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    ports:
      - "8123:8123"
    volumes:
      - /mnt/data/homeassistant:/config
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=Europe/Madrid
    depends_on:
      - mosquitto
      - zigbee2mqtt
    networks:
      - iot

networks:
  iot:
    driver: bridge
```

Orden de arranque (por el usuario MQTT):

```bash
docker compose up -d mosquitto
source .env
docker exec mosquitto mosquitto_passwd -b /mosquitto/config/passwd "$MQTT_USER" "$MQTT_PASSWORD"
docker compose restart mosquitto
docker compose up -d
```

!!! note "Mosquitto no publica puertos"
    Solo es accesible dentro de la red `iot`. Home Assistant se conecta a `mosquitto:1883`.

## 5. Zigbee2MQTT: ajustes tras el asistente

La primera vez que entras en `http://IP_UBUNTU:8088`, Zigbee2MQTT 2.x muestra un **asistente de onboarding**. Al terminar, revisa a mano su configuración (el archivo lo escribe el contenedor con otro usuario, por eso `sudo`):

```bash
sudo nano /mnt/data/zigbee2mqtt/data/configuration.yaml
```

```yaml
serial:
  adapter: ezsp          # ver gotcha de firmware
  port: /dev/serial/by-id/usb-ITEAD_SONOFF_Zigbee_3.0_USB_Dongle_Plus_V2_XXXXXXXX-if00
  baudrate: 115200
frontend:
  enabled: true          # el asistente lo deja en false
  port: 8080
homeassistant:
  enabled: true          # formato objeto, no booleano
onboarding: false        # si queda en true, vuelve al asistente en cada reinicio
```

```bash
docker compose restart zigbee2mqtt
```

### Gotchas de Zigbee2MQTT

| Síntoma | Causa y solución |
|---|---|
| `EZSP protocol version (8) is not supported by Host [13-19]` | El Sonoff V2 de fábrica trae firmware EZSP v8, incompatible con el driver `ember`. Usa `adapter: ezsp` (funciona, aunque avisa de que está obsoleto). La solución definitiva es flashear un firmware NCP ≥ 7.4.x en el dongle. |
| `homeassistant must be object` | La variable `ZIGBEE2MQTT_CONFIG_HOMEASSISTANT=true` (booleano) choca con el formato `{enabled: true}` que escribe el asistente. Quita la variable y corrige el YAML. |
| `ERR_CONNECTION_REFUSED` en el puerto 8088 | El asistente puso `frontend.enabled: false`. No es un problema de firewall. |
| Vuelve al asistente tras reiniciar | `onboarding: false` en el YAML. |
| *Permission denied* al editar el YAML | Edítalo con `sudo` desde el host. |

## 6. Home Assistant

1. Entra en `http://IP_UBUNTU:8123` y crea el usuario administrador (`Usuario_HomeAssistant` / `Contraseña_HomeAssistant`).
2. **Ajustes → Dispositivos y servicios → Añadir integración → MQTT:** servidor `mosquitto`, puerto `1883`, usuario y contraseña del `.env`.
3. Los dispositivos emparejados en Zigbee2MQTT aparecen solos (descubrimiento MQTT).

Para que funcione detrás de Nginx Proxy Manager, añade a `/mnt/data/homeassistant/configuration.yaml`:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - IP_UBUNTU
    - 172.16.0.0/12
```

```bash
docker compose restart homeassistant
```

!!! warning "Sin esto, error 400"
    Home Assistant rechaza con *400 Bad Request* las peticiones que llegan por un proxy que no está en `trusted_proxies`.

## 7. Triángulo de configuración

Dos servicios web:

| Dominio | Destino | Websockets |
|---|---|---|
| `ha.lan` | `http://IP_UBUNTU:8123` | ✅ obligatorio |
| `zigbee.lan` | `http://IP_UBUNTU:8088` | ✅ obligatorio |

Para cada uno: *rewrite* en AdGuard, certificado con Step-CA (`step ca certificate "ha.lan" ha.crt ha.key`), Proxy Host en NPM con Force SSL, y registro en Homepage y Uptime Kuma.

!!! danger "El frontend de Zigbee2MQTT no tiene login por defecto"
    Quien llegue a `zigbee.lan` puede emparejar o eliminar dispositivos. Protégelo con `frontend.auth_token` en su `configuration.yaml`, o no lo publiques en NPM y accede solo por `IP_UBUNTU:8088` cuando lo necesites.

---

## 8. Persianas Zigbee: calibrar el tiempo de recorrido

Los módulos de persiana genéricos Tuya (p. ej. Lonsonho QS-Zigbee-C03) calculan el % de apertura **por tiempo**, no por posición real. Traen `calibration_time: 60` de fábrica, que casi nunca coincide con tu persiana, así que el % mostrado no es fiable.

!!! warning "El interruptor *Calibration* del frontend no aprende el tiempo"
    Hacer un ciclo completo con *Calibration = ON* no cambia `calibration_time`, y el frontend no permite editarlo.

Lo que sí funciona:

1. Manda **abrir** del todo y cronometra hasta que el motor se para solo en el tope. Repite para **cerrar**.
2. Escribe ese tiempo (en segundos) directamente por MQTT:

```bash
cd ~/docker/home-assistant && source .env
docker exec mosquitto mosquitto_pub -h localhost -u "$MQTT_USER" -P "$MQTT_PASSWORD" \
  -t "zigbee2mqtt/NOMBRE_DISPOSITIVO/set" -m '{"calibration_time": SEGUNDOS}'
```

El nombre del dispositivo admite espacios y tildes (`zigbee2mqtt/habitación 1/set`). Confírmalo en el log de Zigbee2MQTT buscando `calibration_time`.

## 9. Emparejamiento y problemas de alcance

- **Emparejar:** activa *Permitir unión* en Zigbee2MQTT solo mientras emparejas, pon el dispositivo en modo *pairing* y sigue el log en tiempo real (`docker logs -f zigbee2mqtt`). Debe aparecer `device_joined` y *Successfully interviewed*.
- **Timeouts en los comandos** (`Timeout after 10000ms`) aunque el dispositivo esté unido: el enlace de radio no es fiable (distancia, paredes gruesas). Un LQI puntual aceptable no garantiza un enlace estable. En este orden:
    1. **Alargador USB** para el dongle (apartado 1).
    2. **Repetidor Zigbee:** cualquier enchufe inteligente Zigbee alimentado siempre, a mitad de camino. Actúa como *router* y añade un salto.
    3. **Canal:** evita solapes con el WiFi 2,4 GHz. Los canales Zigbee 15, 20 y 25 no se pisan con los WiFi 1, 6 y 11.
- **Un dispositivo no se detecta en absoluto:** si en el log no aparece ningún intento de unión, o está fuera de alcance, o no está realmente en modo *pairing* (revisa el procedimiento de reset del fabricante).
