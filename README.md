# ip-camera-rtsp-viewer

Streaming en vivo de una camara IP con salida RTSP (Tapo, Reolink, Hikvision, Dahua, UniFi Protect, etc.) accesible por navegador via web, sin depender de la app movil del fabricante ni de su nube. Usa [go2rtc](https://github.com/AlexxIT/go2rtc) como puente RTSP -> navegador.

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

2. Averigua la IP local de la camara (app del fabricante, o lista DHCP de tu router).

3. Averigua el path RTSP correcto para tu marca/modelo. Cada fabricante usa un esquema distinto:
   - **Tapo**: `/stream1` (calidad alta, puede ir en HEVC) o `/stream2` (calidad baja, H.264)
   - **Reolink**: `/h264Preview_01_main` o `/h264Preview_01_sub`
   - **Hikvision**: `/Streaming/Channels/101` (principal) o `/Streaming/Channels/102` (secundario)
   - **Dahua**: `/cam/realmonitor?channel=1&subtype=0`
   - **Generico ONVIF**: usa una herramienta como [ONVIF Device Manager](https://sourceforge.net/projects/onvifdm/) para descubrir el path exacto si el fabricante no lo documenta.

   Si tienes dudas, prueba primero con `ffprobe` (ver seccion Diagnostico) antes de tocar la config de go2rtc.

4. Clona el repo y crea tu `.env`:

   ```bash
   git clone git@github.com:toninoes/ip-camera-rtsp-viewer.git
   cd ip-camera-rtsp-viewer
   cp .env.example .env
   ```

5. Edita `.env` con tus valores reales (usuario, contrasena, IP, path). Este archivo nunca se sube al repositorio, esta en `.gitignore`.

6. Ajusta el path RTSP en `go2rtc.yaml` si tu camara no es una Tapo (el valor por defecto usa `/stream2`, especifico de Tapo).

7. Levanta el contenedor:

   ```bash
   docker compose up -d
   ```

8. Abre `http://localhost:1984` en el navegador. Deberias ver el stream `camara_principal` listado, con reproduccion en vivo.

## Codec: H.264 vs HEVC/H.265

Muchas camaras de gama alta envian su stream principal en **HEVC/H.265** cuando la resolucion es 4K o similar. Los navegadores no soportan HEVC de forma nativa sobre WebRTC, asi que go2rtc falla en timeout al intentar servirlo, no es un problema de red ni de credenciales.

La solucion mas simple es usar el stream secundario de la camara (menor resolucion, casi siempre H.264 por defecto). Si necesitas resolucion completa, hay dos alternativas:
- Bajar la calidad del stream principal a H.264 desde la app/interfaz del fabricante, si esa opcion existe.
- Forzar transcodificacion en go2rtc con el prefijo `ffmpeg:` y `#video=h264` en la URL del stream (consume CPU del host).

## Diagnostico

Si el contenedor no muestra imagen, aisla el problema antes de tocar la config de go2rtc:

```bash
# 1. Conectividad de red
ping -c 3 <IP_CAMARA>

# 2. Puerto RTSP abierto
nc -zv <IP_CAMARA> 554

# 3. Sesion RTSP y codec real que envia la camara
ffprobe -rtsp_transport tcp "rtsp://usuario:contrasena@<IP_CAMARA>:554/<PATH_RTSP>"
```

Si `ffprobe` muestra `hevc` en vez de `h264`, ese es el origen del problema tipico con timeouts, no sigas revisando red ni credenciales.

## Persistencia tras reinicio

El contenedor usa `restart: unless-stopped`, por lo que se relevanta solo tras un reinicio del sistema, siempre que Docker este habilitado a nivel de systemd:

```bash
sudo systemctl enable docker
```

Si detienes el contenedor manualmente (`docker compose stop`), no se relevantara solo en el siguiente arranque hasta que lo inicies tu de nuevo, ese es el comportamiento esperado de `unless-stopped`.

## Multiples camaras o ubicaciones

Este repo esta pensado para una camara por despliegue. Si tienes camaras en ubicaciones distintas (redes separadas), clona el repo por separado en cada host y usa un `.env` distinto en cada uno, no compartas IP ni credenciales entre despliegues aunque sea el mismo modelo de camara.

Si en cambio quieres varias camaras en el mismo host, anade streams adicionales directamente en `go2rtc.yaml`:

```yaml
streams:
  camara_salon:
    - rtsp://${CAMARA_SALON_USER}:${CAMARA_SALON_PASSWORD}@${CAMARA_SALON_IP}:554/stream2
  camara_entrada:
    - rtsp://${CAMARA_ENTRADA_USER}:${CAMARA_ENTRADA_PASSWORD}@${CAMARA_ENTRADA_IP}:554/h264Preview_01_sub
```

## Acceso remoto

Este setup solo sirve el stream en la red local del host donde corre Docker. Para acceso remoto sin abrir puertos al router (no recomendado por superficie de exposicion), usa un tunel tipo [Tailscale](https://tailscale.com/) apuntando al puerto `1984`.

## Seguridad

- No expongas el puerto `554` (RTSP) ni `1984` (API de go2rtc) directamente a internet.
- `.env` contiene credenciales en texto plano, revisa que este en `.gitignore` antes del primer commit.
- Usa credenciales RTSP distintas por camara/ubicacion.

## Camaras probadas

- TP-Link Tapo C260 (funciona con `/stream2`, `/stream1` requiere transcodificacion por HEVC)

Si pruebas este setup con otra marca/modelo, un PR anadiendo el path RTSP correspondiente a esta seccion es bienvenido.

## Licencia

MIT
