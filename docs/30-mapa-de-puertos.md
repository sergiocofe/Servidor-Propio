# 30 · Mapa de puertos

Referencia rápida de **qué puerto usa cada servicio** en el host (`IP_UBUNTU`). Consúltala antes de desplegar algo nuevo para no provocar conflictos.

!!! note "Solo NPM se usa desde fuera"
    Todos los servicios web se usan a través de Nginx Proxy Manager (`https://servicio.lan`, puerto 443). Los puertos de esta tabla son el **destino interno** de cada Proxy Host.

## Infraestructura y seguridad

| Servicio | Puerto(s) host | Dominio | Sección |
|---|---|---|---|
| SSH | 22222/tcp | — | [01](01-proxmox-y-vm.md) |
| DNS (AdGuard Home) | 53/tcp+udp | — | [09](09-adguard-home.md) |
| AdGuard Home (panel) | 3000 | `adguard.lan` | [09](09-adguard-home.md) |
| Nginx Proxy Manager | 80, 443, 81 (panel) | `nginx.lan` | [03](03-nginx-proxy-manager.md) |
| Portainer | 9443, 8000 | `portainer.lan` | [02](02-docker-portainer.md) |
| Step-CA | 9000 | — | [04](04-ssl-step-ca.md) |
| Authentik | 9200 (HTTP), 9201 (HTTPS) | `authentik.lan` | [12](12-authentik.md) |
| CrowdSec (API local) | 8091 (solo `127.0.0.1`) | — | [17](17-crowdsec-ufw.md) |
| Proxy del socket Docker | — (solo red `socket_proxy`) | — | [02](02-docker-portainer.md) |

## Monitorización

| Servicio | Puerto host | Dominio | Sección |
|---|---|---|---|
| Uptime Kuma | 3001 | `uptime.lan` | [05](05-uptime-kuma.md) |
| Netdata | 19999 | `netdata.lan` | [06](06-netdata.md) |
| Dozzle | 8888 | `logs.lan` | [07](07-dozzle.md) |
| Homepage | 3005 | `home.lan` | [08](08-homepage.md) |

## Aplicaciones

| Servicio | Puerto host | Dominio | Sección |
|---|---|---|---|
| Vaultwarden | 8080 | `vault.lan` | [13](13-vaultwarden.md) |
| MeTube | 8081 | `metube.lan` | [21](21-metube.md) |
| Nextcloud | 8082 | `nextcloud.lan` | [14](14-nextcloud.md) |
| Paperless-ngx | 8083 | `paperless.lan` | [23](23-paperless.md) |
| Stirling PDF | 8084 | `pdf.lan` | [25](25-stirling-pdf.md) |
| Wallos | 8085 | `wallos.lan` | [26](26-wallos.md) |
| Grocy | 8086 | `grocy.lan` | [27](27-grocy.md) |
| IT-Tools | 8087 | `tools.lan` | [28](28-it-tools.md) |
| Zigbee2MQTT | 8088 | `zigbee.lan` | [29](29-home-assistant.md) |
| qBittorrent | 8090 (web), 6881/tcp+udp | `torrent.lan` | [20](20-qbittorrent.md) |
| Jellyfin | 8096 | `jellyfin.lan` | [19](19-jellyfin.md) |
| Home Assistant | 8123 | `ha.lan` | [29](29-home-assistant.md) |
| Immich | 2283 | `immich.lan` | [24](24-immich.md) |
| n8n | 5678 | `n8n.lan` | [15](15-n8n.md) |
| Audiobookshelf | 13378 | `books.lan` | [22](22-audiobookshelf.md) |

## Fuera del servidor

| Servicio | Dónde | Puerto |
|---|---|---|
| WireGuard (hub) | VM pública | 51820/udp |

## Conflictos resueltos

| Conflicto | Solución |
|---|---|
| CrowdSec (8080 por defecto) vs Vaultwarden | CrowdSec → **8091** |
| Authentik (9000/9443 por defecto) vs Step-CA (9000) y Portainer (9443) | Authentik → **9200/9201** |
| qBittorrent (web 8080) vs Vaultwarden | publicado en **8090** (`WEBUI_PORT=8080` interno) |
| AdGuard Home necesita el 53 | red `host` |
| `python3 -m http.server 8080` falla con *Address already in use* | usa otro puerto libre (8099) |

!!! tip "Comprobar qué está ocupado"
    ```bash
    sudo ss -tulpn | grep LISTEN
    docker ps --format 'table {{.Names}}\t{{.Ports}}'
    ```
