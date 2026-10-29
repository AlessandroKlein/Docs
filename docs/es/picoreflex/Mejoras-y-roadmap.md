---
tags:
  - picoreflex
  - roadmap
  - mejoras
---

# Hoja de Ruta de Mejoras — PicoReflex

> **Base:** v0.9.5 · **Fecha:** 2026-10-29

> La hoja de ruta completa (con "qué / cómo / por qué" y referencias oficiales del
> Pico SDK) vive en el repositorio:
> [`docs/ROADMAP.md`](https://github.com/AlessandroKlein/PicoReflex/blob/main/docs/ROADMAP.md).
> Esta página resume el estado y lo pendiente.

## ✅ Completado hasta v0.9.0

- **Watchdog hardware** (P0) — `watchdog_enable(8000 ms)` + `watchdog_update()` en ambos bucles.
- **Debounce de pulsadores por PIO** (P1) — `button_debounce.pio`, una state machine por botón.
- **Credenciales Wi-Fi configurables desde la web** (P0) — persistidas en Flash (v0.2.0).
- **Reconexión Wi-Fi en caliente (STA→AP)** (P0) — `sta_link_watchdog()`, backoff exponencial.
- **Servidor DHCP del AP** + **captive portal** (DNS) — `src/dhcpserver.c`.
- **mDNS `picoreflex.local`** (P1) — `LWIP_MDNS_RESPONDER` + `pico_lwip_mdns`.
- **OTA A/B con bootloader** (P1) — `POST /ota/upload` + CRC32 + *rollback*.
- **Soporte SH1106** (P1) — selección en runtime desde `/display` y persistencia.
- **Calibración de la batería** (P1) — `BatteryCalibration` configurable desde `/battery`.
- **FreeRTOS (SMP, 2 núcleos)** (v0.5.0) — `game_task` + `network_task`.
- **Modo Simón Dice** (v0.6.0) — `GameMode::SIMON_SAYS`.
- **LEDs WS2812 opcionales** (v0.6.0) — macro `PICOREFLEX_WS2812`.
- **Autenticación PIN + rate-limiting** (v0.7.0) — `WEB_PIN` + cookie/Bearer.
- **Clave del AP configurable/persistida** — por defecto `12345678`, editable desde `/wifi`.
- **CI endurecido** (v0.7.0) — `cppcheck` + reporte de tamaño (`target size`).
- **Modo dormir configurable** (v0.9.0) — `sleep_enabled` + `idle_sleep_min` desde `/display`, persistido en Flash.
- **Fix cuelgue `cyw43_arch_init()`** (v0.9.0) — `TCPIP_THREAD_STACKSIZE 1024` en `lwipopts.h`.
- **Fix bootloader "USB no reconocido"** (v0.9.0) — `jump_to_app()` con IRQ habilitadas.
- **Clave del AP fija `12345678`** (v0.9.0) — revert del RUID, editable desde `/wifi`.

## ✅ Completado en v0.9.1–v0.9.5

- **Reconexión AP→STA en caliente** (v0.9.2) — delay de asentamiento de 200 ms + fix spam "STA activa".
- **PANIC Wi-Fi `MEMP_SYS_TIMEOUT`** (v0.9.4) — `MEMP_NUM_SYS_TIMEOUT 32` en `lwipopts.h`.
- **Afinidad de núcleos** (v0.9.5) — `game_task`→Core 0, `network_task`→Core 1 (fix del reinicio al guardar config).
- **Display OLED** (v0.9.5) — doble charge-pump (SSD1306+SH1106), auto-test `0xA5` al arrancar y fix del SH1106 en blanco (addressing por página).
- **Simón Dice acelera la secuencia** (v0.9.5) — cada vez más rápido a medida que crece.
- **Persistencia del modo/dificultad del juego** (v0.9.5) — `game_mode`/`game_difficulty`/`seed`/`round_seconds`/`stimulus_*` en Flash (`PERSISTENT_VERSION` 7).
- **Arranque con cualquier botón** (v0.9.5) — reemplaza el disparador de dos botones simultáneos.

## Pendiente

### P2 (deseable)

- **Más modos de juego** (duelo por Wi-Fi, reacción por niveles).
- **Autonomía estimada** en el panel web (horas restantes).
- **Manejo de ausencia del display** I2C (saltear `show()` si no hay ACK).
- **Reentrancia de Flash** (mutex global si ambos núcleos escriben).
- **Tamaño de imagen / memoria** (revisar particionado con OTA activa).

### P3 (opcional / investigación)

- **TLS (HTTPS)** con `pico_mbedtls` para el panel en modo STA.
- **Firmado de firmware** (secure boot) si se comercializa.
- **Multijugador** (2 Pico W por Wi-Fi), **gráficos en OLED**, **registro de eventos exportable**.

## Referencias

- [Pico SDK (índice Doxygen)](https://www.raspberrypi.com/documentation/pico-sdk/index_doxygen.html)
- [pico-examples](https://github.com/raspberrypi/pico-examples)
- [lwIP (mDNS, raw API)](https://www.nongnu.org/lwip/2_1_x/)
