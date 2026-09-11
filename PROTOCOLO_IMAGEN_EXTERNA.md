# Protocolo Imagen Externa (monitor hardware → DMD)

Receptor de frames en tiempo real para el DMD. Permite que una aplicación externa
(Python en Linux/Windows, servicio de monitorización, etc.) pinte la matriz
interrumpiendo la reproducción de GIFs de la SD, con retorno automático a GIFs.

- **Transporte:** TCP, puerto **8889**, conexión persistente (keep-alive recomendado)
- **Formato:** RGB565 little-endian, 128×32 px, sin compresión, sin ACK (fire & forget)
- **Firmware:** v3.1.2 (`firmware/3.1.2/`) — `ESTADO_IMAGEN_EXTERNA`
- **Validado en hardware:** ESP32-D0WD-V3 (WROOM, sin PSRAM), `PANEL_CHAIN=2`,
  12 FPS sostenidos, latencia de pintado < 5 ms/frame

## 1. Formato de frame

Cada frame son **8196 bytes** consecutivos:

```
Offset  Tamaño  Contenido
0       4       Cabecera: 0xAA 0x55 0x80 0x20
                    [0]=0xAA magic, [1]=0x55 magic,
                    [2]=0x80 ancho (128), [3]=0x20 alto (32)
4       8192    Payload: 128 × 32 píxeles RGB565 little-endian, row-major
                    píxel = R[4:0] G[5:0] B[4:0], LSB primero
                    píxel (x,y) → bytes [(y*128+x)*2, (y*128+x)*2+1]
```

Conversión RGB888 → RGB565:

```python
rgb565 = ((r >> 3) << 11) | ((g >> 2) << 5) | (b >> 3)
frame_bytes = rgb565.astype('<u2').tobytes()   # NumPy, little-endian
```

## 2. Comportamiento del receptor

| Evento | Acción del ESP32 |
|--------|------------------|
| Frame válido completo | Interrumpe GIFs (`ESTADO_IMAGEN_EXTERNA`), pinta ambos paneles, `imagenExterna=1` |
| Frames continuos | Repinta cada frame (~12 FPS validados) |
| Sin frames durante `IMAGE_TIMEOUT` (defecto 1000 ms) | Vuelve a GIFs (`saliendoAGifs`), `imagenExterna=0` |
| Cabecera inválida | Descarta cliente, espera nueva conexión |
| Nueva conexión TCP | Reemplaza a la anterior (el último que conecta manda) |
| Stream arcade (:8888) simultáneo | **La imagen externa tiene prioridad**: el modo arcade no pinta mientras `imagenExternaActiva` |

Notas de implementación:

- Recepción **incremental**: el receptor acumula cabecera + payload por partes en
  cada sondeo (robusto ante segmentación TCP y ventana reducida de lwIP,
  ~5760 B/sondeo observados). El emisor puede enviar de golpe, por trozos o con
  frames encadenados en la misma conexión.
- Pintado: `drawRGBBitmap(0,0,…)` + `drawRGBBitmap(128,0,…)` (ambos paneles,
  sin `fillScreen` previo) + `flipDMABuffer()`. Sin conversión por píxel en el
  ESP32: el buffer ya llega en formato nativo de display.
- Ancho de banda a 12 FPS: 8196 × 12 ≈ **96 KB/s**.
- Sin autenticación ni ACK en v1.

## 3. Configuración

`config.ini` (sección `[IMAGEN_EXTERNA]`, persistente en SD):

```ini
[IMAGEN_EXTERNA]
# Timeout sin frames para volver a GIFs (ms): 200 a 5000
IMAGE_TIMEOUT=1000
```

API REST (puerto 80):

- `GET /status` → incluye `"imagenExterna": 0/1`
- `GET /config` → incluye `"IMAGE_TIMEOUT": <ms>`
- `POST /config` → acepta `"IMAGE_TIMEOUT"` (clamp 200–5000 ms)

## 4. Ejemplo mínimo de emisor (Python)

```python
import socket, time

IP, PORT = "192.168.1.66", 8889
W, H = 128, 32
HEADER = bytes([0xAA, 0x55, W, H])

def rgb888_to_rgb565_le(pixels):  # pixels: lista de (r,g,b)
    out = bytearray(len(pixels) * 2)
    for i, (r, g, b) in enumerate(pixels):
        v = ((r >> 3) << 11) | ((g >> 2) << 5) | (b >> 3)
        out[2*i] = v & 0xFF
        out[2*i+1] = (v >> 8) & 0xFF
    return bytes(out)

s = socket.create_connection((IP, PORT), timeout=3)
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
period = 1.0 / 12
while True:
    t0 = time.perf_counter()
    s.sendall(HEADER + rgb888_to_rgb565_le(render_frame()))  # tu render 128x32
    dt = time.perf_counter() - t0
    if period - dt > 0:
        time.sleep(period - dt)
# Al cerrar el socket, el DMD vuelve a GIFs en ~IMAGE_TIMEOUT ms.
```

Recomendaciones para el emisor: conexión persistente con reconexión exponencial,
timing con `time.perf_counter()`, render con Cairo + NumPy para 12 FPS estables.

## 5. Diagnóstico

| Síntoma | Causa probable |
|---------|----------------|
| `curl http://<IP>/status` no tiene campo `imagenExterna` | Firmware anterior sin el endpoint: reflashear v3.1.2+ |
| Conexión a :8889 rechazada/timeout | `imageServer` solo arranca con WiFi conectado en el arranque; revisar `IP local:` en serial |
| Serial no muestra `[IMAGEN] Servidor :8889 listo` | El binario flasheado no incluye el receptor |
| Frame aceptado pero no se pinta / `imagenExterna` siempre 0 | Payload incompleto o cabecera inválida; ver `[IMAGEN] Cabecera invalida` en serial |
| Parpadeo negro en un panel | Regresión del pintado dual: verificar los dos `drawRGBBitmap` (x=0 y x=128) sin `fillScreen` previo |

Trazas del receptor (serial 115200):

```
[IMAGEN] Servidor :8889 listo (RGB565 128x32).
[IMAGEN] Stream externo activo, interrumpiendo GIFs.
[IMAGEN] Cabecera invalida, descartando cliente.
[IMAGEN] Timeout: volviendo a GIFs.
```

## 6. Compilación / flasheo (referencia verificada)

El sketch usa estructura Arduino IDE (multi-`.ino` concatenados); `pio run` no
sirve. Compilado y flasheado con `arduino-cli` + core `esp32:esp32@2.0.17`:

```bash
export PATH="$HOME/bin:$PATH"
# Copia sin espacios (el esptool del core 2.0.17 falla con espacios en ruta):
rm -rf /tmp/flash && mkdir -p /tmp/flash/RetroPixelLED312 && \
cp "firmware/3.1.2/Retro Pixel LED lite 3.1.2/Retro Pixel LED lite 3.1.2.ino" \
   /tmp/flash/RetroPixelLED312/RetroPixelLED312.ino && \
cp firmware/3.1.2/WebRoutes.ino firmware/3.1.2/fuente8pt7b_*.h /tmp/flash/RetroPixelLED312/ && \
arduino-cli compile --upload -p /dev/ttyUSB0 \
  --fqbn "esp32:esp32:esp32:PartitionScheme=huge_app,FlashSize=4M" /tmp/flash/RetroPixelLED312
```

Librerías (Arduino Library Manager): `ESP32 HUB75 LED MATRIX PANEL DMA Display`,
`AnimatedGIF`, `ESP32FtpServer`, `ArduinoJson` 6.x, `IRremote`, `Adafruit GFX Library`.
