# ip-camera-rtsp-viewer

Streaming en vivo de una camara IP con salida RTSP (Tapo, Reolink, Hikvision, Dahua, UniFi Protect, etc.) accesible por navegador via web, sin depender de la app movil del fabricante ni de su nube. Usa [MediaMTX](https://github.com/bluenviron/mediamtx) como servidor multimedia y puente RTSP -> WebRTC.

Este proyecto utiliza [MediaMTX](https://github.com/bluenviron/mediamtx), un proyecto independiente distribuido bajo licencia MIT. Este repositorio no incluye ni modifica su codigo fuente.

## Requisitos

- Docker + Docker Compose v2
- Una camara IP con RTSP habilitado y accesible por red local
- `ffmpeg` (opcional, solo para diagnostico, ver mas abajo)

## Setup

1. Habilita RTSP en la camara y crea credenciales especificas para ese protocolo (no reutilices las de la cuenta cloud del fabricante). El sitio exacto varia por marca:
   - **Tapo**: App Tapo -> Camara -> Ajustes -> Avanzado -> Cuenta de la camara.
   - **Reolink**: App Reolink -> Camara -> Configuracion avanzada -> Red -> RTSP/ONVIF.
   - **Hikvision/Dahua**: interfaz web de la camara -> Configuracion -> Red -> Protocolos avanzados -> ONVIF/RTSP.
   - **UniFi Protect**: Panel de administracion -> Camara -> Configuracion -> Avanzado -> RTSP.

2. Averigua la IP local de la camara (app del fabricante, o lista DHCP de tu router). Lo ideal es asignarle una IP estatica dentro de tu red local para que no cambie y evitar tener que actualizar la configuracion. En el caso analizado, una TP-Link Tapo C260, esta reserva se puede establecer desde la propia app de Tapo para Android:

<img src="img/camera-ip.png" alt="Configuracion de IP estatica en la app Tapo" width="30%" style="display: block; margin: 0 auto;">


3. Averigua el path RTSP correcto para tu marca/modelo. Cada fabricante usa un esquema distinto:
   - **Tapo**: `/stream1` (calidad alta, puede ir en HEVC) o `/stream2` (calidad baja, H.264)
   - **Reolink**: `/h264Preview_01_main` o `/h264Preview_01_sub`
   - **Hikvision**: `/Streaming/Channels/101` (principal) o `/Streaming/Channels/102` (secundario)
   - **Dahua**: `/cam/realmonitor?channel=1&subtype=0`
   - **Generico ONVIF**: usa una herramienta como [ONVIF Device Manager](https://sourceforge.net/projects/onvifdm/) para descubrir el path exacto si el fabricante no lo documenta.

   Si tienes dudas, prueba primero con `ffprobe` (ver seccion Diagnostico) antes de tocar la configuracion de MediaMTX.

4. Clona el repo y crea tu `.env`:

   ```bash
   git clone git@github.com:toninoes/ip-camera-rtsp-viewer.git
   cd ip-camera-rtsp-viewer
   cp .env.example .env
   ```

5. Edita `.env` con tus valores reales (usuario, contrasena, IP, path). Este archivo nunca se sube al repositorio, esta en `.gitignore`.

6. Ajusta `CAMERA_RTSP_PATH` en `.env` si tu camara no es una Tapo (el valor por defecto usa `/stream2`, especifico de Tapo).

7. Levanta el contenedor:

   ```bash
   docker compose up -d --force-recreate
   ```

8. Abre [http://localhost:8889/camara](http://localhost:8889/camara) en el navegador para ver el stream WebRTC.

En el caso analizado, una TP-Link Tapo C260, el resultado es una reproduccion en vivo directamente en el navegador, con imagen a pantalla completa, sonido y la marca temporal de la propia camara:

<img src="img/working.png" alt="Reproduccion en vivo de la camara IP" width="80%" style="display: block; margin: 0 auto;">


## Codec: H.264 vs HEVC/H.265

Muchas camaras de gama alta envian su stream principal en **HEVC/H.265** cuando la resolucion es 4K o similar. Los navegadores no soportan HEVC de forma nativa sobre WebRTC, asi que MediaMTX no puede entregarlo directamente al navegador, no es un problema de red ni de credenciales.

La solucion mas simple es usar el stream secundario de la camara (menor resolucion, casi siempre H.264 por defecto). Si necesitas resolucion completa, hay dos alternativas:
- Bajar la calidad del stream principal a H.264 desde la app/interfaz del fabricante, si esa opcion existe.
- Usar el stream secundario H.264. MediaMTX no transcodifica video por si mismo; si necesitas convertir HEVC a H.264, añade un componente externo como FFmpeg (consume CPU del host).

## Configuracion

La configuracion principal esta en `mediamtx.yml`. La URL RTSP se construye en `docker-compose.yml` usando las variables de `.env` y se pasa a MediaMTX mediante `MTX_PATHS_CAMARA_SOURCE`, que sobrescribe el valor base `source: publisher` del path `camara`.

### Camara y path RTSP

Edita `.env` y recrea el contenedor despues de cualquier cambio:

```dotenv
CAMERA_USER=usuario_rtsp
CAMERA_PASSWORD=contrasena_rtsp
CAMERA_IP=192.168.1.200
CAMERA_RTSP_PATH=/stream2
```

Ejemplos de `CAMERA_RTSP_PATH`:

- Tapo: `/stream1` o `/stream2`
- Reolink: `/h264Preview_01_main` o `/h264Preview_01_sub`
- Hikvision: `/Streaming/Channels/101` o `/Streaming/Channels/102`
- Dahua: `/cam/realmonitor?channel=1&subtype=0`

### Reproduccion y puertos

Con `network_mode: host`, MediaMTX escucha directamente en el host:

| Puerto | Funcion |
|---|---|
| `8889/tcp` | Pagina y señalizacion WebRTC: `http://localhost:8889/camara` |
| `8189/udp` | Trafico ICE/WebRTC del navegador |
| `8888/tcp` | HLS, disponible en `http://localhost:8888/camara/index.m3u8` |
| `9997/tcp` | API de control, desactivada por defecto |

WebRTC es la opcion recomendada para el navegador. Si la red bloquea UDP, puedes probar HLS, aunque normalmente tendra mas latencia. Los servidores RTSP, RTMP, SRT y MoQ estan desactivados porque este proyecto no los utiliza.

### Transporte RTSP hacia la camara

MediaMTX elige automaticamente el transporte RTSP. Si la camara funciona mejor con TCP, añade esta opcion al path en `mediamtx.yml`:

```yaml
paths:
   camara:
      source: publisher
      sourceOnDemand: true
      rtspTransport: tcp
```

### Logs y diagnostico

Para aumentar temporalmente el detalle de los logs, añade `MTX_LOGLEVEL: debug` dentro de `environment` en `docker-compose.yml`. Despues consulta:

```bash
docker compose logs -f mediamtx
```

Vuelve a `info` o elimina la variable cuando termines el diagnostico.

### API de control

La API permite consultar y administrar paths, pero no proporciona un panel visual. Para habilitarla, añade lo siguiente a `docker-compose.yml`:

```yaml
environment:
   MTX_API: "yes"
```

Despues de recrear el contenedor, quedara disponible en `http://localhost:9997`. No expongas ese puerto a internet: con `network_mode: host` tambien puede quedar accesible desde la red local.

### Multiples camaras

Para varias camaras, añade un path por camara en `mediamtx.yml`. Los paths adicionales deben usar una URL RTSP completa o sus propias variables `MTX_PATHS_*_SOURCE` en `docker-compose.yml`:

```yaml
paths:
   camara_salon:
      source: rtsp://usuario:contrasena@192.168.1.201:554/stream2
      sourceOnDemand: true
```

Evita guardar credenciales reales en `mediamtx.yml`; usa variables de entorno y comprueba que `.env` no se versiona.

## Comandos utiles

Gestion basica del contenedor:

```bash
# Levantar el contenedor (en segundo plano)
docker compose up -d

# Ver estado y desde cuando lleva corriendo
docker compose ps

# Parar el contenedor (no se relevanta solo hasta que lo arranques tu de nuevo)
docker compose stop

# Recrear tras un cambio en mediamtx.yml, docker-compose.yml o .env
docker compose up -d --force-recreate

# Reiniciar forzando recreacion completa del contenedor (si restart no basta)
docker compose down && docker compose up -d

# Eliminar el contenedor y liberar el nombre (no borra imagenes ni volumenes)
docker compose down
```

Logs:

```bash
# Logs en vivo, siguiendo la salida
docker compose logs -f mediamtx

# Ultimas 100 lineas sin seguir en vivo
docker compose logs --tail=100 mediamtx

# Logs con timestamp explicito (util para correlacionar con cortes de red)
docker compose logs -f -t mediamtx

# Logs desde un momento concreto
docker compose logs --since 30m mediamtx
```

Inspeccion del contenedor:

```bash
# Variables de entorno realmente cargadas dentro del contenedor
# (confirma que .env se esta inyectando bien, sin exponerlas en el yaml)
docker exec mediamtx env | grep CAMERA

# Entrar a una shell dentro del contenedor
docker exec -it mediamtx sh

# Ver la configuracion que MediaMTX esta usando en runtime
docker exec mediamtx cat /mediamtx.yml

# Uso de recursos del contenedor (CPU/memoria), util si activaste transcodificacion ffmpeg
docker stats mediamtx
```

## Diagnostico

Si el contenedor no muestra imagen, aisla el problema en este orden, de red hacia arriba, no empieces por la configuracion de MediaMTX:

```bash
# 1. Conectividad de red basica
ping -c 3 <IP_CAMARA>

# 2. Puerto RTSP abierto y alcanzable
nc -zv <IP_CAMARA> 554

# 3. Sesion RTSP real y codec que envia la camara
ffprobe -rtsp_transport tcp "rtsp://usuario:contrasena@<IP_CAMARA>:554/<PATH_RTSP>"

# 4. Si no tienes ffmpeg instalado en el host, usa un contenedor desechable
docker run --rm --network host jrottenberg/ffmpeg \
  -rtsp_transport tcp -i "rtsp://usuario:contrasena@<IP_CAMARA>:554/<PATH_RTSP>" \
  -t 2 -f null -
```

Interpretacion de errores tipicos en los logs de MediaMTX:

| Sintoma en `docker compose logs` | Causa probable | Que revisar |
|---|---|---|
| `i/o timeout` al conectar con la camara | La camara no responde en la URL RTSP, o el path, usuario o contrasena son incorrectos | Ejecuta `ffprobe` y revisa el `source` de `mediamtx.yml` |
| `i/o timeout` en una URL RTSP normal, pero `nc` confirma el puerto abierto | Stream en codec no soportado (HEVC) o RTSP no habilitado realmente en la camara pese a tener credenciales creadas | Ejecuta `ffprobe` y revisa el campo `Video:` de la salida |
| `401 Unauthorized` o `Unauthorized` en los logs | Usuario o contrasena RTSP incorrectos, o contrasena con caracteres especiales sin URL-encodear | Prueba con una contrasena solo alfanumerica para descartar el problema de encoding |
| `connection refused` en vez de timeout | RTSP no habilitado en la camara, o puerto distinto a 554 | Revisa Ajustes avanzados de la camara y confirma el puerto real |
| Contenedor no aparece como `Up` en `docker compose ps` | Error de sintaxis en `mediamtx.yml` o `.env` no encontrado | `docker compose logs mediamtx` deberia mostrar el error de parseo al arrancar |
| Video se ve pero va a saltos o se corta | Ancho de banda insuficiente en la red local, o demasiadas sesiones RTSP concurrentes contra la misma camara | Cierra la app movil del fabricante mientras pruebas, muchas camaras limitan a 1-2 clientes RTSP simultaneos |

Si `ffprobe` muestra `hevc` en vez de `h264` en el campo `Video:`, ese es el origen del problema tipico con timeouts, no sigas revisando red ni credenciales.

## Persistencia tras reinicio

El contenedor usa `restart: unless-stopped`, por lo que se relevanta solo tras un reinicio del sistema, siempre que Docker este habilitado a nivel de systemd:

```bash
sudo systemctl enable docker
sudo systemctl is-enabled docker   # debe devolver "enabled"
```

Si detienes el contenedor manualmente (`docker compose stop`), no se relevantara solo en el siguiente arranque hasta que lo inicies tu de nuevo, ese es el comportamiento esperado de `unless-stopped`.

## Multiples camaras o ubicaciones

Este repo esta pensado para una camara por despliegue. Si tienes camaras en ubicaciones distintas (redes separadas), clona el repo por separado en cada host y usa un `.env` distinto en cada uno, no compartas IP ni credenciales entre despliegues aunque sea el mismo modelo de camara.

Si en cambio quieres varias camaras en el mismo host, anade paths adicionales directamente en `mediamtx.yml`:

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

Este setup solo sirve el stream en la red local del host donde corre Docker. Para acceso remoto sin abrir puertos al router (no recomendado por superficie de exposicion), usa un tunel tipo [Tailscale](https://tailscale.com/) apuntando al puerto `8889`.

## Seguridad

- No expongas el puerto `554` (RTSP de la camara) ni `8889` (WebRTC de MediaMTX) directamente a internet.
- `.env` contiene credenciales en texto plano, revisa que este en `.gitignore` antes del primer commit.
- Usa credenciales RTSP distintas por camara/ubicacion.

## Camaras probadas

- TP-Link Tapo C260 (funciona con `/stream2`, `/stream1` requiere transcodificacion por HEVC)

Si pruebas este setup con otra marca/modelo, un PR anadiendo el path RTSP correspondiente a esta seccion es bienvenido.

## Licencia

MIT
