# EUNO Autopilot — instalación manual

Piloto automático de código abierto para embarcaciones pequeñas, sobre ESP32-S3.
Este documento explica cómo instalarlo desde cero: el firmware en la placa y la
aplicación en el teléfono.

**Versión actual: 1.3.12** (firmware y aplicación llevan siempre el mismo número).

---

## Antes de empezar: seguridad

Esto mueve un timón. Antes de probarlo en el agua:

- El piloto **no puede encenderse sin brújula**. Si el sensor no responde, el
  firmware bloquea el motor y avisa (`FAILSAFE,COMPASS_INIT`). Es intencionado.
- El mando manual **siempre tiene prioridad**: cualquier corrección manual
  suspende la ruta automática.
- Las cartas de la aplicación son **una ayuda visual, no sustituyen las cartas
  náuticas oficiales**. Las profundidades vienen de un modelo público (EMODnet),
  con una resolución de unos 115 metros.
- La primera prueba con el motor conectado hazla amarrado, no navegando.

---

## 1. Firmware en la placa

### Qué hace falta

| | |
|---|---|
| Placa | ESP32-S3 con **16 MB** de flash |
| Esquema de particiones | **`app3M_fat9M_16MB`** — imprescindible |
| Núcleo Arduino | `esp32:esp32` **3.1.0** |

El esquema de particiones no es un detalle: el que viene por defecto reserva
1,25 MB para la aplicación y el firmware no cabe. Además hacen falta los 9,9 MB
de FAT para los registros de navegación, las rutas y los puntos de pesca.

### Opción A — grabar el binario ya compilado (lo más rápido)

Descarga [`euno-firmware-completo.bin`](euno-firmware-completo.bin) de este
mismo repositorio. Contiene el gestor de arranque, la tabla de particiones y la
aplicación: se graba de una vez en la dirección 0.

```bash
esptool.py --chip esp32s3 --port COM4 --baud 921600 write_flash 0x0 euno-firmware-completo.bin
```

Sustituye `COM4` por tu puerto (en Linux o macOS será algo como
`/dev/ttyUSB0` o `/dev/cu.usbserial-0001`).

Este archivo es una instantánea de la versión 1.3.12 y no se actualiza en cada
release. No importa: **solo hace falta para la primera grabación**. A partir de
ahí la placa se actualiza sola desde la aplicación, que siempre ofrece la última.

Si la placa ya tenía otro firmware, conviene borrarla antes:

```bash
esptool.py --chip esp32s3 --port COM4 erase_flash
```

### Opción B — compilar desde el código fuente

Bibliotecas necesarias (Gestor de bibliotecas del IDE de Arduino):

- Adafruit ICM20X
- Adafruit Unified Sensor
- Adafruit BusIO
- TinyGPSPlus
- WebSockets (de Markus Sattler / Links2004)
- NimBLE-Arduino 1.4.x

Con `arduino-cli`, desde la carpeta del sketch:

```bash
arduino-cli compile --fqbn esp32:esp32:fri3d_2024_esp32s3:PartitionScheme=app3M_fat9M_16MB .
arduino-cli upload -p COM4 --fqbn esp32:esp32:fri3d_2024_esp32s3:PartitionScheme=app3M_fat9M_16MB .
```

Desde el IDE de Arduino: elige la placa ESP32-S3, y en **Herramientas →
Partition Scheme** selecciona `app3M_fat9M_16MB`.

### Conexiones

| Señal | Pin |
|---|---|
| Sensor IMU — SDA | GPIO **8** |
| Sensor IMU — SCL | GPIO **9** |
| GPS — TX del módulo | GPIO **16** |
| Motor — RPWM (extensión) | GPIO **3** |
| Motor — LPWM (retracción) | GPIO **46** |

Sensor: **ICM-20948** por I2C (dirección 0x68). El GPS es cualquier receptor
u-blox o compatible a **9600 baudios** que emita NMEA estándar; funcionan tanto
las frases `$GP...` como las `$GN...` de los módulos multiconstelación.

Alimenta el módulo IMU a 3,3 V. Si tu placa expone el pin **NCS**, debe estar en
alto para que hable por I2C; muchas placas ya lo llevan fijado.

---

## 2. La aplicación

Descarga [`EUNOapp.apk`](EUNOapp.apk) e instálala. Android pedirá permiso para
instalar aplicaciones de origen desconocido: hay que concedérselo al navegador o
al gestor de archivos con el que abras el archivo.

A partir de ahí **la aplicación se actualiza sola**: al arrancar comprueba si hay
una versión nueva y la ofrece.

No hace falta la aplicación para usar el piloto: la placa sirve la misma interfaz
en `http://192.168.4.1` desde cualquier navegador. Lo que solo tiene la
aplicación es la carta de costas detallada, que ocupa demasiado para la memoria
de la placa.

---

## 3. Primera puesta en marcha

1. Alimenta la placa. Crea una red Wi-Fi propia:
   **`EunoAutopilot`**, contraseña **`password`** (es la de fábrica; conviene
   cambiarla antes de usarla en serio).
2. Conecta el teléfono a esa red y abre la aplicación.
3. En la pestaña **Setup**, ejecuta `CAL MAG` y gira el sensor lentamente en
   todas las direcciones durante 30 segundos, **sin desplazarlo por la
   habitación**: hay que girarlo sobre sí mismo. Si lo paseas, mides el hierro
   del edificio en lugar del campo terrestre.
4. Con el GPS al aire libre, espera a que aparezcan satélites.

La red de la placa no da acceso a internet. Es normal, y por eso la descarga de
cartas y de actualizaciones se hace **antes**, conectado a una red normal.

---

## 4. Si algo no funciona

El firmware trae herramientas de diagnóstico. Se envían por el puerto serie a
115200 baudios, o desde la consola de la aplicación:

| Comando | Para qué |
|---|---|
| `$PEUNO,CMD,DIAG=I2C` | Quién responde en el bus I2C y **qué chip es realmente**. Prueba las dos combinaciones de pines y lee el identificador: distingue un ICM-20948 auténtico de un MPU reetiquetado, que es frecuente en los módulos económicos. |
| `$PEUNO,CMD,DIAG=GPS` | Prueba seis velocidades en ambos pines y dice cuál produce NMEA válido. Sirve cuando el GPS no da señales de vida. |
| `$PEUNO,CMD,DIAG=TILT` | Lecturas en vivo: aceleraciones, campo magnético, los cuatro rumbos y los caracteres recibidos del GPS. |
| `$PEUNO,CMD,LOG=LIST` | Registros de navegación guardados en la placa. |

Dos comprobaciones rápidas con `DIAG=TILT`:

- el campo magnético debe tener un módulo de unos **45-50 µT** en Europa; si da
  casi cero, la brújula no está leyendo;
- el contador `chars` debe **crecer** si el GPS está conectado, aunque todavía no
  haya satélites.

---

## Licencia

© 2025 Yari Gabbai — CC BY-NC 4.0: uso no comercial, citando al autor.
