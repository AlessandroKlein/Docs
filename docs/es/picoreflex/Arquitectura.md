# Arquitectura del Proyecto — PicoReflex

## 1. Visión general

PicoReflex es un **dispositivo de entrenamiento de reflejos** construido sobre la
**Raspberry Pi Pico W** (RP2040 + radio inalámbrica CYW43439), programado en
**C/C++ con el Pico SDK oficial 2.3.0** (toolchain `arm-none-eabi`, CMake +
Ninja). **No usa Arduino ni MicroPython.**

Consta de 5 LEDs y 5 pulsadores: se enciende un LED y el jugador debe presionar
el botón correspondiente en el menor tiempo posible. El firmware añade semilla
determinista (secuencias reproducibles), servidor web embebido, medición de
batería, FreeRTOS SMP, watchdog, mDNS, actualización OTA A/B y autenticación por
PIN.

## 2. Diagrama de bloques

```text
                        +--------------------------------------------+
                        |         Raspberry Pi Pico W                |
                        |         (RP2040 + CYW43439)                |
                        +--------------------------------------------+
                          |                                   |
             +------------+-----------+            +----------+------------+
             |   game_task (juego)    |            | network_task (red/UI) |
             +------------------------+            +-----------------------+
             | game_logic  (motor)    |  estado    | web_server (HTTP)     |
             | audio       (buzzer)   | compartido | web_ui    (HTML/CSS)  |
             | led_driver  (LEDs)     |  critical  | display_driver (OLED) |
             | button_debounce (PIO)  | _section + | battery_monitor (ADC) |
             | flash_storage (Flash)  |  queue     | cyw43_arch / lwIP     |
             +---+---+---+------------+            | captive_portal (DNS)  |
                 |   |   |             |           | mDNS + DHCP + OTA     |
                 v   v   v             v           +-----------------------+
             GPIO 2..6  GPIO 7..11   GPIO 16            |              |
              (LEDs)   (pulsadores)  (buzzer)      GPIO 14/15     CYW43 Wi-Fi
                                                        |        (STA/SoftAP)
                                                   I2C1 0x3C
                                               SSD1306/SH1106

        Batería LiPo 1S ── 10k/22k ── ADC0 (GPIO26)  ← divisor resistivo
```

![Diagrama de bloques del sistema](img/Diagrama%20de%20bloques.svg)

El juego y la red corren como **dos tareas de FreeRTOS (SMP)** que el scheduler
reparte entre los dos núcleos del RP2040: `game_task` y `network_task`. No hay
afinidad fija de núcleo (equivalen conceptualmente al antiguo Core 0/Core 1 del
firmware pre-FreeRTOS).

## 3. Estructura de archivos

```text
PicoReflex/
├── CMakeLists.txt               # Build (Pico SDK 2.3.0, PICO_BOARD=pico_w)
├── lwipopts.h                   # lwIP: NO_SYS=0, LWIP_MDNS_RESPONDER, LWIP_IGMP
├── FreeRTOS_Kernel_import.cmake # Integración FreeRTOS SMP
├── ws2812.pio                   # PIO: tira NeoPixel (GPIO28)
├── button_debounce.pio          # PIO: debounce por hardware (una SM por botón)
├── linker/
│   └── pico_flash_region.ld     # Mapa de Flash para OTA A/B (bootloader+app+staging)
├── include/
│   ├── pin_defs.hpp             # Mapa de pines (ver docs/Idea.md §4)
│   ├── config.hpp               # Config/estadísticas + API de estado compartido
│   ├── game_logic.hpp           # Motor del juego + PRNG (xorshift)
│   ├── audio.hpp                # Buzzer PWM
│   ├── display_driver.hpp       # OLED SSD1306/SH1106 (I2C1)
│   ├── battery_monitor.hpp      # ADC batería + calibración (BatteryCalibration)
│   ├── flash_storage.hpp        # Persistencia en Flash + metadatos OTA
│   ├── web_server.hpp           # Servidor HTTP + Wi-Fi (network_task)
│   ├── captive_portal.hpp       # DNS 53 → 192.168.4.1 (modo AP)
│   ├── led_driver.hpp           # LEDs GPIO o WS2812 (PIO)
│   ├── ota.h                    # Metadatos OTA (CRC32, slots A/B)
│   └── font5x7.hpp              # Fuente 5x7 (generada por tools/gen_font.py)
├── src/
│   ├── main.cpp                 # Arranque FreeRTOS + watchdog + lanzar tareas
│   ├── shared_state.cpp         # critical_section + queue (estado compartido)
│   ├── game_logic.cpp           # Motor del juego (game_task)
│   ├── audio.cpp                # Buzzer
│   ├── display_driver.cpp       # OLED SSD1306/SH1106
│   ├── battery_monitor.cpp      # ADC + curva de descarga
│   ├── flash_storage.cpp        # Flash (récords, config, OTA, clave AP)
│   ├── web_server.cpp           # HTTP + Wi-Fi + display + mDNS + OTA (network_task)
│   ├── web_ui.cpp               # Frontend HTML/CSS embebido
│   ├── captive_portal.cpp       # DNS captive portal
│   ├── dhcpserver.c             # DHCP del AP
│   ├── led_driver.cpp           # GPIO o WS2812 (PIO)
│   ├── ota.c                    # CRC32 + metadatos (compartido con bootloader)
│   └── bootloader.c             # Bootloader de segundo nivel (OTA A/B)
└── docs/                        # Documentación técnica (este sitio la refleja)
```

## 4. Flujo del juego (`game_task`)

1. `game_logic_init()` prepara el PRNG (semilla determinista si `seed != 0`).
2. Bucle de `game_logic_run()`:
   - Procesa eventos de control (`START_ROUND`, `STOP_ROUND`, `CONFIG_CHANGED`…).
   - Enciende un LED al azar y mide el tiempo de reacción (microsegundos).
   - Registra el resultado en `stats_record()` (historial de 20 reacciones).
   - Alimenta el watchdog (`watchdog_update()`).
3. Los pulsadores se leen por **PIO** (`button_debounce.pio`): cada botón tiene
   una máquina de estados que notifica al CPU por IRQ sin latencia de rebote.

## 5. Flujo de red (`network_task`)

1. `web::init()` ejecuta `cyw43_arch_init()` (modo `sys_freertos`, `NO_SYS=0`)
   y arranca el display, la batería y lwIP.
2. `web::run()`:
   - Modo **STA**: conecta a la red configurada; si cae el enlace,
     `sta_link_watchdog()` reintenta y, tras agotar los reintentos, pasa a **AP**.
   - Modo **AP**: levanta SoftAP WPA2 + DHCP (`dhcpserver.c`) + DNS captive
     portal (`captive_portal.cpp`) + mDNS.
   - En STA anuncia `picoreflex.local` por **mDNS** (lwIP `mdns`).
   - Sirve el panel web (HTTP :80) con autenticación **PIN** y **rate-limiting**.
   - Recibe `POST /ota/upload` y graba la imagen candidata en el slot de staging.

## 6. Sincronización inter-tarea

- `shared_state.cpp` protege `GameConfig`/`GameStats`/`SystemStatus` con un
  `critical_section_t` (accesos muy cortos).
- Los eventos de control viajan de `network_task` a `game_task` por una
  `queue_t` (`pico/util/queue.h`).
- Flags `volatile bool` para el protocolo de dormancia (`g_sleep_request`,
  `g_sleep_ready`) y la persistencia (`g_persist_done`).

## 7. Esquema de hardware (resumen)

| Periférico            | Pin GPIO       | Notas                                           |
|-----------------------|----------------|-------------------------------------------------|
| LED 0..4              | 2..6           | Ánodo por GPIO con resistencia ~220–470 Ω       |
| Botón 0..4            | 7..11          | Pull-up interno, debounce por PIO               |
| OLED SDA / SCL        | 14 / 15        | I2C1 400 kHz, 0x3C/0x3D (SSD1306 0.96" / SH1106 1.3") |
| Buzzer pasivo         | 16             | PWM (onda cuadrada)                             |
| Batería               | 26 (ADC0)      | Divisor externo 10 k / 22 k (no VSYS/3)         |
| LED onboard           | CYW43          | Estado de red (fijo = STA, parpadeo = AP)       |
| NeoPixel (opcional)   | 28             | `PICOREFLEX_WS2812=1` (PIO)                     |

El **display OLED** soporta dos controladores, ambos 128×64 por I2C1 en
`0x3C`/`0x3D`: **SSD1306** (0.96") y **SH1106** (1.3"). El controlador se
selecciona en runtime desde `/display` (`?controller=0|1`) y se **persiste en
Flash**; la macro `PICOREFLEX_DISPLAY_CONTROLLER` solo fija el valor por defecto.

> **Referencia:** [Pinout oficial Raspberry Pi Pico W (A4, RP-008315-DS-1)](https://pip-assets.raspberrypi.com/categories/686-raspberry-pi-pico-w/documents/RP-008315-DS-1-PicoW-A4-Pinout.pdf)

## 8. Estimación de consumo

- **Activo (STA + juego):** el Wi-Fi domina el consumo (~60–90 mA con la radio
  encendida); el OLED y los LEDs suman unos pocos mA.
- **Dormancia (configurable):** tras `idle_sleep_min` minutos de inactividad
  (1..60, por defecto 3) y con el Wi-Fi apagado, `game_task` pide dormir,
  `network_task` apaga el Wi-Fi (`cyw43_arch_deinit`) y el RP2040 entra en modo
  **DORMANT** (< 2 mA). Un pulsador lo despierta. Configurable desde `/display`
  (Habilitar/Deshabilitar + minutos); `sleep_enabled=false` desactiva el sueño.
- **Batería:** la lectura del ADC se convierte a % con una **curva de descarga
  calibrable** (3 puntos + compensación de carga), editable desde `/battery`.

## 9. Actualización OTA (A/B)

El mapa de Flash reserva un **bootloader de segundo nivel** en `0x10000000`, el
**slot de aplicación** en `0x10010000` (896 KB) y un **slot de staging** en
`0x100F0000`. `POST /ota/upload` recibe el binario (CRC32 verificado), lo graba
en staging y el commit cambia el puntero de arranque; si el arranque falla, hay
**rollback** automático a la imagen anterior. Ver `Compilacion.md`.
