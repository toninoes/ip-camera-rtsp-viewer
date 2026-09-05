# ip-camera-rtsp-viewer

Streaming en vivo de una cámara IP con salida RTSP (Tapo, Reolink, Hikvision, Dahua, UniFi Protect, etc.) accesible desde el navegador vía web, sin depender de la aplicación móvil del fabricante ni de su nube. Usa [MediaMTX](https://github.com/bluenviron/mediamtx) como servidor multimedia y puente RTSP -> WebRTC.

Este proyecto utiliza [MediaMTX](https://github.com/bluenviron/mediamtx), un proyecto independiente distribuido bajo licencia MIT. Este repositorio no incluye ni modifica su código fuente.

## Flujo de señal

La cámara entrega el vídeo a MediaMTX mediante RTSP. MediaMTX actúa como proxy y adapta ese stream al protocolo que utiliza el cliente:

```mermaid
flowchart LR
   camera[Cámara IP] -->|RTSP :554| mediamtx[MediaMTX]
   mediamtx -->|WebRTC :8889| browser[Navegador]
   mediamtx -->|HLS :8888| hls[Cliente HLS opcional]
```

En este proyecto, MediaMTX es el cliente RTSP de la cámara y el navegador es el cliente WebRTC de MediaMTX. Por eso `rtsp: false` en `mediamtx.yml` no desactiva la entrada RTSP: solo desactiva el servidor RTSP de salida de MediaMTX, que escucharía en el puerto `8554` y no se necesita aquí.

La URL de la cámara se construye con las variables de `.env` y se inyecta mediante `MTX_PATHS_CAMARA_SOURCE`. El path `camara` utiliza `sourceOnDemand: true`, por lo que MediaMTX solo abre la conexión RTSP cuando existe un cliente reproduciendo el stream.

## Requisitos

- Docker + Docker Compose v2
- Una cámara IP con RTSP habilitado y accesible por la red local
- `ffmpeg` (opcional, solo para diagnóstico; véase más abajo)

## Setup

1. Habilita RTSP en la cámara y crea credenciales específicas para ese protocolo (no reutilices las de la cuenta en la nube del fabricante). El sitio exacto varía según la marca:
   - **Tapo**: aplicación Tapo -> Cámara -> Ajustes -> Avanzado -> Cuenta de la cámara.
   - **Reolink**: aplicación Reolink -> Cámara -> Configuración avanzada -> Red -> RTSP/ONVIF.
   - **Hikvision/Dahua**: interfaz web de la cámara -> Configuración -> Red -> Protocolos avanzados -> ONVIF/RTSP.
   - **UniFi Protect**: panel de administración -> Cámara -> Configuración -> Avanzado -> RTSP.

2. Averigua la IP local de la cámara (aplicación del fabricante o lista DHCP de tu router). Lo ideal es asignarle una IP estática dentro de tu red local para que no cambie y evitar tener que actualizar la configuración. En el caso analizado, una TP-Link Tapo C260, esta reserva se puede establecer desde la propia aplicación de Tapo para Android:

<img src="img/camera-ip.png" alt="Configuración de IP estática en la aplicación Tapo" width="30%" style="display: block; margin: 0 auto;">


3. Averigua el path RTSP correcto para tu marca o modelo. Cada fabricante usa un esquema distinto:
   - **Tapo**: `/stream1` (calidad alta, puede ir en HEVC) o `/stream2` (calidad baja, H.264)
   - **Reolink**: `/h264Preview_01_main` o `/h264Preview_01_sub`
   - **Hikvision**: `/Streaming/Channels/101` (principal) o `/Streaming/Channels/102` (secundario)
   - **Dahua**: `/cam/realmonitor?channel=1&subtype=0`
   - **Genérico ONVIF**: usa una herramienta como [ONVIF Device Manager](https://sourceforge.net/projects/onvifdm/) para descubrir el path exacto si el fabricante no lo documenta.

   Si tienes dudas, prueba primero con `ffprobe` (véase la sección Diagnóstico) antes de tocar la configuración de MediaMTX.

4. Clona el repositorio y crea tu `.env`:

   ```bash
   git clone git@github.com:toninoes/ip-camera-rtsp-viewer.git
   cd ip-camera-rtsp-viewer
   cp .env.example .env
   ```

5. Edita `.env` con tus valores reales (usuario, contraseña, IP y path). Este archivo nunca se sube al repositorio y está incluido en `.gitignore`.

6. Ajusta `CAMERA_RTSP_PATH` en `.env` si tu cámara no es una Tapo (el valor predeterminado usa `/stream2`, específico de Tapo).

7. Levanta el contenedor:

   ```bash
   docker compose up -d --force-recreate
   ```

8. Abre [http://localhost:8889/camara](http://localhost:8889/camara) en el navegador para ver el stream WebRTC.

En el caso analizado, una TP-Link Tapo C260, el resultado es una reproducción en vivo directamente en el navegador, con imagen a pantalla completa, sonido y la marca temporal de la propia cámara:

<img src="img/working.png" alt="Reproducción en vivo de la cámara IP" width="80%" style="display: block; margin: 0 auto;">


## Códec: H.264 frente a HEVC/H.265

Muchas cámaras de gama alta envían su stream principal en **HEVC/H.265** cuando la resolución es 4K o similar. Los navegadores no soportan HEVC de forma nativa sobre WebRTC, así que MediaMTX no puede entregarlo directamente al navegador; no es un problema de red ni de credenciales.

La solución más simple es usar el stream secundario de la cámara (menor resolución, casi siempre H.264 por defecto). Si necesitas resolución completa, hay dos alternativas:
- Bajar la calidad del stream principal a H.264 desde la aplicación o interfaz del fabricante, si esa opción existe.
- Usar el stream secundario H.264. MediaMTX no transcodifica vídeo por sí mismo; si necesitas convertir HEVC a H.264, añade un componente externo como FFmpeg (consume CPU del host).

## Configuración

La configuración principal está en `mediamtx.yml`. La URL RTSP se construye en `docker-compose.yml` usando las variables de `.env` y se pasa a MediaMTX mediante `MTX_PATHS_CAMARA_SOURCE`, que sobrescribe el valor base `source: publisher` del path `camara`.

### Cámara y path RTSP

Edita `.env` y recrea el contenedor después de cualquier cambio:

```dotenv
CAMERA_USER=usuario_rtsp
CAMERA_PASSWORD=password_rtsp
CAMERA_IP=192.168.1.200
CAMERA_RTSP_PATH=/stream2
```

Ejemplos de `CAMERA_RTSP_PATH`:

- Tapo: `/stream1` o `/stream2`
- Reolink: `/h264Preview_01_main` o `/h264Preview_01_sub`
- Hikvision: `/Streaming/Channels/101` o `/Streaming/Channels/102`
- Dahua: `/cam/realmonitor?channel=1&subtype=0`

### Reproducción y puertos

Con `network_mode: host`, MediaMTX escucha directamente en el host:

| Puerto | Funcion |
|---|---|
| `8889/tcp` | Página y señalización WebRTC: `http://localhost:8889/camara` |
| `8189/udp` | Tráfico ICE/WebRTC del navegador |
| `8888/tcp` | Página HLS: `http://localhost:8888/camara` |
| `9997/tcp` | API de control, desactivada por defecto |

WebRTC es la opción recomendada para el navegador. Si la red bloquea UDP, puedes probar HLS, aunque normalmente tendrá más latencia. Los servidores RTSP, RTMP, SRT y MoQ están desactivados porque este proyecto no los utiliza.

### Transporte RTSP hacia la cámara

MediaMTX elige automáticamente el transporte RTSP. Si la cámara funciona mejor con TCP, añade esta opción al path en `mediamtx.yml`:

```yaml
paths:
   camara:
      source: publisher
      sourceOnDemand: true
      rtspTransport: tcp
```

### Logs y diagnóstico

Para aumentar temporalmente el detalle de los logs, añade `MTX_LOGLEVEL: debug` dentro de `environment` en `docker-compose.yml`. Después consulta:

```bash
docker compose logs -f mediamtx
```

Vuelve a `info` o elimina la variable cuando termines el diagnóstico.

### API de control

La API permite consultar y administrar paths, pero no proporciona un panel visual. Para habilitarla, añade lo siguiente a `docker-compose.yml`:

```yaml
environment:
   MTX_API: "yes"
```

Después de recrear el contenedor, quedará disponible en `http://localhost:9997`. No expongas ese puerto a internet: con `network_mode: host` también puede quedar accesible desde la red local.

### Múltiples cámaras

Para varias cámaras, añade un path por cámara en `mediamtx.yml`. Los paths adicionales deben usar una URL RTSP completa o sus propias variables `MTX_PATHS_*_SOURCE` en `docker-compose.yml`:

```yaml
paths:
   camara_salon:
      source: rtsp://usuario:password@192.168.1.201:554/stream2
      sourceOnDemand: true
```

Evita guardar credenciales reales en `mediamtx.yml`; usa variables de entorno y comprueba que `.env` no se incluye en el control de versiones.

## Comandos útiles

Gestión básica del contenedor:

```bash
# Levantar el contenedor (en segundo plano)
docker compose up -d

# Ver estado y desde cuando lleva corriendo
docker compose ps

# Parar el contenedor (no se reinicia automáticamente hasta que lo arranques de nuevo)
docker compose stop

# Recrear tras un cambio en mediamtx.yml, docker-compose.yml o .env
docker compose up -d --force-recreate

# Reiniciar forzando recreacion completa del contenedor (si restart no basta)
docker compose down && docker compose up -d

# Eliminar el contenedor y liberar el nombre (no borra imágenes ni volúmenes)
docker compose down
```

Logs:

```bash
# Logs en vivo, siguiendo la salida
docker compose logs -f mediamtx

# Últimas 100 líneas sin seguir en vivo
docker compose logs --tail=100 mediamtx

# Logs con timestamp explicito (util para correlacionar con cortes de red)
docker compose logs -f -t mediamtx

# Logs desde un momento concreto
docker compose logs --since 30m mediamtx
```

Inspeccion del contenedor:

```bash
# Variables de entorno realmente cargadas dentro del contenedor
# (confirma que .env se está inyectando correctamente, sin exponerlas en el YAML)
docker exec mediamtx env | grep CAMERA

# Entrar a una shell dentro del contenedor
docker exec -it mediamtx sh

# Ver la configuración que MediaMTX está usando en tiempo de ejecución
docker exec mediamtx cat /mediamtx.yml

# Uso de recursos del contenedor (CPU/memoria), util si activaste transcodificacion ffmpeg
docker stats mediamtx
```

## Diagnóstico

Si el contenedor no muestra imagen, aísla el problema en este orden, desde la red hacia arriba. No empieces por la configuración de MediaMTX:

```bash
# 1. Conectividad de red básica
ping -c 3 <IP_CAMARA>

# 2. Puerto RTSP abierto y alcanzable
nc -zv <IP_CAMARA> 554

# 3. Sesión RTSP real y códec que envía la cámara
ffprobe -rtsp_transport tcp "rtsp://usuario:password@<IP_CAMARA>:554/<PATH_RTSP>"

# 4. Si no tienes FFmpeg instalado en el host, usa un contenedor desechable
docker run --rm --network host jrottenberg/ffmpeg \
  -rtsp_transport tcp -i "rtsp://usuario:password@<IP_CAMARA>:554/<PATH_RTSP>" \
  -t 2 -f null -
```

Interpretación de errores típicos en los logs de MediaMTX:

| Sintoma en `docker compose logs` | Causa probable | Que revisar |
|---|---|---|
| `i/o timeout` al conectar con la cámara | La cámara no responde en la URL RTSP, o el path, usuario o contraseña son incorrectos | Ejecuta `ffprobe` y revisa el `source` de `mediamtx.yml` |
| `i/o timeout` en una URL RTSP normal, pero `nc` confirma el puerto abierto | Stream con códec no compatible (HEVC) o RTSP no habilitado realmente en la cámara pese a tener credenciales creadas | Ejecuta `ffprobe` y revisa el campo `Video:` de la salida |
| `401 Unauthorized` o `Unauthorized` en los logs | Usuario o contraseña RTSP incorrectos, o contraseña con caracteres especiales sin codificar en la URL | Prueba con una contraseña solo alfanumérica para descartar el problema de codificación |
| `connection refused` en vez de timeout | RTSP no habilitado en la cámara, o puerto distinto a 554 | Revisa los ajustes avanzados de la cámara y confirma el puerto real |
| El contenedor no aparece como `Up` en `docker compose ps` | Error de sintaxis en `mediamtx.yml` o `.env` no encontrado | `docker compose logs mediamtx` debería mostrar el error de análisis al arrancar |
| El vídeo se ve, pero va a saltos o se corta | Ancho de banda insuficiente en la red local, o demasiadas sesiones RTSP simultáneas contra la misma cámara | Cierra la aplicación móvil del fabricante mientras pruebas; muchas cámaras limitan a 1 o 2 clientes RTSP simultáneos |

Si `ffprobe` muestra `hevc` en vez de `h264` en el campo `Video:`, ese es el origen del problema típico con los tiempos de espera agotados; no sigas revisando la red ni las credenciales.

## Persistencia tras un reinicio

El contenedor usa `restart: unless-stopped`, por lo que se reinicia automáticamente tras un reinicio del sistema, siempre que Docker esté habilitado mediante systemd:

```bash
sudo systemctl enable docker
sudo systemctl is-enabled docker   # debe devolver "enabled"
```

Si detienes el contenedor manualmente (`docker compose stop`), no se reiniciará automáticamente en el siguiente arranque hasta que lo inicies de nuevo; ese es el comportamiento esperado de `unless-stopped`.

## Múltiples cámaras o ubicaciones

Este repositorio está pensado para una cámara por despliegue. Si tienes cámaras en ubicaciones distintas (redes separadas), clona el repositorio por separado en cada host y usa un `.env` distinto en cada uno; no compartas IP ni credenciales entre despliegues, aunque sea el mismo modelo de cámara.

Si, en cambio, quieres varias cámaras en el mismo host, añade paths adicionales directamente en `mediamtx.yml`:

```yaml
paths:
   camara_salon:
      source: rtsp://${CAMARA_SALON_USER}:${CAMARA_SALON_PASSWORD}@${CAMARA_SALON_IP}:554/stream2
      sourceOnDemand: true
   camara_entrada:
      source: rtsp://${CAMARA_ENTRADA_USER}:${CAMARA_ENTRADA_PASSWORD}@${CAMARA_ENTRADA_IP}:554/h264Preview_01_sub
      sourceOnDemand: true
```

## Acceso remoto

Esta configuración solo sirve el stream en la red local del host donde corre Docker. Para acceder de forma remota sin abrir puertos en el router (no recomendado por la superficie de exposición), usa un túnel como [Tailscale](https://tailscale.com/) apuntando al puerto `8889`.

## Seguridad

- No expongas el puerto `554` (RTSP de la cámara) ni `8889` (WebRTC de MediaMTX) directamente a internet.
- `.env` contiene credenciales en texto plano; revisa que esté en `.gitignore` antes del primer commit.
- Usa credenciales RTSP distintas por cámara y ubicación.

## Cámaras probadas

- TP-Link Tapo C260 (funciona con `/stream2`; `/stream1` requiere transcodificación por HEVC)

Si pruebas esta configuración con otra marca o modelo, será bienvenido un PR que añada el path RTSP correspondiente a esta sección.

## Licencia

MIT
