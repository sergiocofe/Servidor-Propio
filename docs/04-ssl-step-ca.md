# 04 · SSL local con Step-CA

Convertimos el servidor en una **Autoridad Certificadora (CA)** propia. Así los servicios `.lan` tendrán el "candado verde" y el tráfico viajará cifrado entre tu equipo y el servidor, sin depender de internet.

---

## 4.1 Instalación de Step-CA

### Paso 1 · Descargar e instalar los paquetes

```bash
wget https://github.com/smallstep/cli/releases/download/v0.24.4/step-cli_0.24.4_amd64.deb
sudo dpkg -i step-cli_0.24.4_amd64.deb

wget https://github.com/smallstep/certificates/releases/download/v0.24.2/step-ca_0.24.2_amd64.deb
sudo dpkg -i step-ca_0.24.2_amd64.deb
```

### Paso 2 · Inicializar la Autoridad Certificadora

```bash
step ca init --name "Nombre_Servidor_CA" \
  --provisioner "Email_Admin" \
  --dns "IP_UBUNTU" \
  --address ":9000"
```

!!! warning "Anota la contraseña"
    Selecciona la opción **Standalone**. Define una contraseña para el certificado raíz (`Contraseña_CA`) y **anótala**: la necesitarás para firmar cada nuevo certificado.

---

## 4.2 Servicio en segundo plano

### Paso 1 · Archivo de contraseña

```bash
echo "Contraseña_CA" > $(step path)/password.txt
chmod 600 $(step path)/password.txt
```

### Paso 2 · Crear el servicio de sistema

```bash
sudo nano /etc/systemd/system/step-ca.service
```

Pega este contenido (ajusta el usuario si es necesario):

```ini
[Unit]
Description=Step CA Service
After=network.target

[Service]
User=Usuario_Ubuntu
Group=Usuario_Ubuntu
ExecStart=/usr/bin/step-ca /home/Usuario_Ubuntu/.step/config/ca.json --password-file=/home/Usuario_Ubuntu/.step/password.txt
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Paso 3 · Activar el servicio

```bash
sudo systemctl daemon-reload      # recarga la configuración del sistema
sudo systemctl enable step-ca     # inicio automático al arrancar
sudo systemctl start step-ca      # arranca el servicio ahora
```

### Paso 4 · Certificados de larga duración

Edita el archivo de configuración:

```bash
nano /home/Usuario_Ubuntu/.step/config/ca.json
```

Añade el bloque `claims` y ajusta el `tls`. La estructura queda así:

```json
{
  "root": "/home/Usuario_Ubuntu/.step/certs/root_ca.crt",
  "federatedRoots": null,
  "crt": "/home/Usuario_Ubuntu/.step/certs/intermediate_ca.crt",
  "key": "/home/Usuario_Ubuntu/.step/secrets/intermediate_ca_key",
  "address": ":9000",
  "insecureAddress": "",
  "dnsNames": ["IP_UBUNTU"],
  "logger": { "format": "text" },
  "db": {
    "type": "badgerv2",
    "dataSource": "/home/Usuario_Ubuntu/.step/db",
    "badgerFileLoadingMode": ""
  },
  "authority": {
    "provisioners": [
      {
        "type": "JWK",
        "name": "Email_Admin",
        "key": {
          "use": "sig",
          "kty": "EC",
          "kid": "<clave_publica_provisioner>",
          "crv": "P-256",
          "alg": "ES256",
          "x": "<generado_por_step_ca_init>",
          "y": "<generado_por_step_ca_init>"
        },
        "encryptedKey": "<NO_PUBLICAR_generado_por_step_ca_init>"
      }
    ],
    "claims": {
      "minTLSCertDuration": "5m",
      "maxTLSCertDuration": "87600h",
      "defaultTLSCertDuration": "87600h"
    }
  },
  "tls": {
    "cipherSuites": [
      "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256",
      "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
    ],
    "minVersion": 1.2,
    "maxVersion": 1.3,
    "renegotiation": false
  }
}
```

!!! danger "No publiques tu ca.json real"
    Los campos `kid`, `x`, `y` y, sobre todo, `encryptedKey` son material criptográfico de **tu** CA y los genera `step ca init` automáticamente. Aquí aparecen como marcadores. **Nunca subas tu `ca.json` real a un repositorio.**

Reinicia el servicio tras editar:

```bash
sudo systemctl restart step-ca
```

---

## 4.3 Generación y uso de certificados

### Paso 1 · Crear los archivos (.crt y .key)

Repite esto para cada servicio que tengas en Nginx Proxy Manager.

```bash
mkdir -p ~/certs && cd ~/certs

step ca certificate "portainer.lan" portainer.crt portainer.key
step ca certificate "adguard.lan"   adguard.crt   adguard.key
step ca certificate "nginx.lan"     nginx.crt     nginx.key
```

### Paso 2 · Descargar los certificados (desde tu PC)

```bash
# Autoridad Certificadora Raíz (común a todos)
scp Usuario_Ubuntu@IP_UBUNTU:~/.step/certs/root_ca.crt .

# Portainer
scp Usuario_Ubuntu@IP_UBUNTU:~/certs/portainer.crt .
scp Usuario_Ubuntu@IP_UBUNTU:~/certs/portainer.key .

# Nginx
scp Usuario_Ubuntu@IP_UBUNTU:~/certs/nginx.crt .
scp Usuario_Ubuntu@IP_UBUNTU:~/certs/nginx.key .

# AdGuard
scp Usuario_Ubuntu@IP_UBUNTU:~/certs/adguard.crt .
scp Usuario_Ubuntu@IP_UBUNTU:~/certs/adguard.key .
```

---

## 4.4 El "candado verde" en Nginx Proxy Manager

### Paso 1 · Subir certificados personalizados

Para cada servicio (`portainer.lan`, `nginx.lan`, `adguard.lan`), en NPM:

1. **SSL Certificates** → **Add SSL Certificate** → **Custom**.
2. **Name:** descriptivo (ej. `SSL_Portainer`).
3. **Certificate Key:** el archivo `.key` correspondiente.
4. **Certificate:** el archivo `.crt` correspondiente.
5. **Intermediate Certificate:** el archivo `root_ca.crt` (el mismo para todos).

### Paso 2 · Activar SSL en los hosts

1. **Proxy Hosts** → edita el servicio.
2. Pestaña **SSL** → selecciona el certificado que subiste.
3. Activa **Force SSL** para obligar a HTTPS.

---

## 4.5 Instalar la confianza en el sistema operativo

Para que desaparezca el aviso de "Conexión no segura" en Windows:

1. Localiza `root_ca.crt` en tu escritorio.
2. Clic derecho → **Instalar certificado**.
3. Elige **Equipo local**.
4. **Colocar todos los certificados en el siguiente almacén** → **Examinar** → **Entidades de certificación raíz de confianza**.
5. Finaliza el asistente y reinicia el navegador.
