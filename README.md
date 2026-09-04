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

8. Abre [http://localhost:1984](http://localhost:1984) en el navegador. Deberias ver el stream `camara_principal` listado, con reproduccion en vivo.

## Codec: H.264 vs HEVC/H.265

Muchas camaras de gama alta envian su stream principal en **HEVC/H.265** cuando la resolucion es 4K o similar. Los navegadores no soportan HEVC de forma nativa sobre WebRTC, asi que go2rtc falla en timeout al intentar servirlo, no es un problema de red ni de credenciales.

La solucion mas simple es usar el stream secundario de la camara (menor resolucion, casi siempre H.264 por defecto). Si necesitas resolucion completa, hay dos alternativas:
- Bajar la calidad del stream principal a H.264 desde la app/interfaz del fabricante, si esa opcion existe.
- Forzar transcodificacion en go2rtc con el prefijo `ffmpeg:` y `#video=h264` en la URL del stream (consume CPU del host).

## Comandos utiles

Gestion basica del contenedor:

```bash
# Levantar el contenedor (en segundo plano)
docker compose up -d

# Ver estado y desde cuando lleva corriendo
docker compose ps

# Parar el contenedor (no se relevanta solo hasta que lo arranques tu de nuevo)
docker compose stop

# Reiniciar tras un cambio en go2rtc.yaml o .env
docker compose restart

# Reiniciar forzando recreacion completa del contenedor (si restart no basta)
docker compose down && docker compose up -d

# Eliminar el contenedor y liberar el nombre (no borra imagenes ni volumenes)
docker compose down
```

Logs:

```bash
# Logs en vivo, siguiendo la salida
docker compose logs -f go2rtc

# Ultimas 100 lineas sin seguir en vivo
docker compose logs --tail=100 go2rtc

# Logs con timestamp explicito (util para correlacionar con cortes de red)
docker compose logs -f -t go2rtc

# Logs desde un momento concreto
docker compose logs --since 30m go2rtc
```

Inspeccion del contenedor:

```bash
# Variables de entorno realmente cargadas dentro del contenedor
# (confirma que .env se esta inyectando bien, sin exponerlas en el yaml)
docker exec go2rtc env | grep CAMERA

# Entrar a una shell dentro del contenedor
docker exec -it go2rtc sh

# Ver la config que go2rtc esta usando en runtime, ya con variables resueltas
docker exec go2rtc cat /config/go2rtc.yaml

# Uso de recursos del contenedor (CPU/memoria), util si activaste transcodificacion ffmpeg
docker stats go2rtc
```

## Diagnostico

Si el contenedor no muestra imagen, aisla el problema en este orden, de red hacia arriba, no empieces por la config de go2rtc:

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

Interpretacion de errores tipicos en los logs de go2rtc:

| Sintoma en `docker compose logs` | Causa probable | Que revisar |
|---|---|---|
| `i/o timeout` en la URL con `#video=h264` o similar | Falta el prefijo `ffmpeg:` delante de la URL para que go2rtc interprete la transcodificacion | Anade `ffmpeg:` al inicio de la URL en `go2rtc.yaml` |
| `i/o timeout` en una URL RTSP normal, pero `nc` confirma el puerto abierto | Stream en codec no soportado (HEVC) o RTSP no habilitado realmente en la camara pese a tener credenciales creadas | Ejecuta `ffprobe` y revisa el campo `Video:` de la salida |
| `401 Unauthorized` o `Unauthorized` en los logs | Usuario o contrasena RTSP incorrectos, o contrasena con caracteres especiales sin URL-encodear | Prueba con una contrasena solo alfanumerica para descartar el problema de encoding |
| `connection refused` en vez de timeout | RTSP no habilitado en la camara, o puerto distinto a 554 | Revisa Ajustes avanzados de la camara y confirma el puerto real |
| Contenedor no aparece como `Up` en `docker compose ps` | Error de sintaxis en `go2rtc.yaml` o `.env` no encontrado | `docker compose logs go2rtc` deberia mostrar el error de parseo al arrancar |
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
