---
tags:
  - picoreflex
  - general
  - pico-w
---

# PicoReflex — Juego de Reacción Rápida Avanzado

> **Tipo:** Guía (portada) | **Estado:** Estable | **Fecha:** 2026-10-29 | **Firmware:** v0.9.6
> **Hardware:** Raspberry Pi Pico W (RP2040 + CYW43439) · **Plataforma:** Pico SDK (oficial) 2.3.0 — C/C++

## 1. Resumen ejecutivo

**PicoReflex** es un **dispositivo de entrenamiento de reflejos** sobre la
**Raspberry Pi Pico W**, programado en **C/C++ con el Pico SDK oficial** (no
Arduino ni MicroPython). El usuario debe presionar el botón que corresponde al
LED encendido en el menor tiempo posible. Combina medición de reacción en
microsegundos, control por semilla determinista, un servidor web embebido de
configuración, medición de batería, conectividad Wi-Fi redundante y actualización
por aire (OTA).

- **5 LEDs + 5 pulsadores** con temporización de reacción en µs y **debounce por
  PIO** (una state machine por botón).
- **PRNG determinista por semilla** (`xorshift32`) para competencias repetibles,
  o modo aleatorio con `pico_rand`. Modos: libre, contrarreloj, muerte súbita y
  **Simón Dice**.
- **FreeRTOS (SMP, 2 núcleos)**: `game_task` (juego) + `network_task` (red/display).
- **Servidor web embebido** (lwIP) con **Station (STA)** y respaldo **SoftAP**
  (`PicoW-ReactionGame`, `192.168.4.1`) + **captive portal** (DNS) + **mDNS**
  (`picoreflex.local` en STA).
- **Autenticación por PIN** y **rate-limiting** en los endpoints de control; clave
  del AP **`12345678` por defecto** (editable desde `/wifi`, persistida en Flash).
- **Medición de batería** por ADC (GPIO26/ADC0, divisor externo 10 k/22 k) con
  curva de descarga **calibrable**.
- **OTA A/B** con bootloader de segundo nivel y *rollback* automático.
- **Watchdog hardware** (reinicio automático ante cuelgues), **modo DORMANT
  configurable** (< 2 mA, desde `/display`) y **almacenamiento no volátil** en Flash.
- **Configuración de juego persistente** (v0.9.5): modo, dificultad, semilla y
  tiempos de estímulo se guardan en Flash y se restauran al arrancar; la ronda
  arranca con **cualquier botón**.

## 2. ¿Qué es?

Es el trabajo práctico final de **Microprocesadores** (UNL — Tecnicatura en
Automatización y Robótica). Implementa en C++ una aplicación completa sobre el
RP2040 con el **Pico SDK oficial 2.3.0** (C/C++, toolchain `arm-none-eabi`):
GPIO, interrupciones, PIO, temporizadores, DMA, PWM, I2C, ADC, Flash, FreeRTOS
SMP, Wi-Fi y pila TCP/IP.

![Qué es PicoReflex](img/Qu%C3%A9%20es.svg)

```text
LEDs (2..6) · Pulsadores (7..11, debounce PIO) · Buzzer (16, PWM) · SSD1306/SH1106 (14/15, I2C1)
          │
          ▼
   PicoReflex — game_task (juego)  ⇄ estado compartido ⇄  network_task (red + display)
          │                                                 │
          ├── ADC0 (GPIO26) → divisor 10k/22k (batería)     ├── cyw43_arch + lwIP (sys_freertos)
          ├── Watchdog hardware                             ├── HTTP + DNS captive portal + mDNS
          └── Flash (récords/config + OTA A/B)              └── OTA (bootloader A/B + staging)
```

## 3. Guías del proyecto

| Documento | Contenido |
|---|---|
| [Compilación y flasheo](Compilacion.md) | Cómo compilar, flashear y usar |
| [Arquitectura](Arquitectura.md) | Bloques, flujos, pines y sincronización |
| [Decisiones](Decisiones.md) | Cada decisión vinculada a la documentación oficial |
| [Referencia API](API.md) | Endpoints HTTP del panel web |
| [Informe técnico](Informe.md) | Límites, solución, conclusión y análisis del ADC |
| [Seguridad](Seguridad.md) | Auditoría de seguridad y hardening |
| [Mejoras y roadmap](Mejoras-y-roadmap.md) | Hoja de ruta de versiones futuras |
| [Dudas](Dudas.md) | Consultas técnicas y su resolución |
| [CHANGELOG](CHANGELOG.md) | Historial de versiones (SemVer) |

## 4. Repositorio de código

- **Código fuente:** [AlessandroKlein/PicoReflex](https://github.com/AlessandroKlein/PicoReflex)
