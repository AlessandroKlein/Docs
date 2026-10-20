---
tags:
  - picoreflex
  - general
  - pico-w
---

# PicoReflex — Juego de Reacción Rápida Avanzado

> **Tipo:** Guía (portada) | **Estado:** Estable | **Fecha:** 2026-10-20 | **Firmware:** v0.1.1
> **Hardware:** Raspberry Pi Pico W (RP2040 + CYW43439) · **Plataforma:** Pico C/C++ SDK 2.3.0

## 1. Resumen ejecutivo

**PicoReflex** es un **dispositivo de entrenamiento de reflejos** basado en la
Raspberry Pi Pico W. El usuario debe presionar el botón que corresponde al LED
encendido en el menor tiempo posible. Combina medición de reacción en
microsegundos, control por semilla determinista, un servidor web embebido de
configuración, medición de batería y conectividad Wi-Fi redundante.

- **5 LEDs + 5 pulsadores** con temporización de reacción en µs (IRQ GPIO +
  `time_us_32`).
- **PRNG determinista por semilla** (`xorshift32`) para competencias repetibles,
  o modo aleatorio con `pico_rand`.
- **Servidor web embebido** (lwIP) con **Station (STA)** y respaldo **SoftAP**
  (`PicoW-ReactionGame`, `192.168.4.1`) + **captive portal** (DNS).
- **Medición de batería** por ADC (VSYS/3) con porcentaje y alerta de baja.
- **Doble núcleo**: Core 0 (juego/IRQ/DMA/PWM/Flash) y Core 1 (red/display).
- **Modo DORMANT** (< 2 mA) y **almacenamiento no volátil** de récords en Flash.

## 2. ¿Qué es?

Es el trabajo práctico final de **Microprocesadores** (UNL — Tecnicatura en
Automatización y Robótica). Implementa en C++ una aplicación completa sobre el
RP2040: GPIO, interrupciones, temporizadores, DMA, PWM, I2C, ADC, Flash,
multinúcleo, Wi-Fi y pila TCP/IP.

```text
LEDs (2..6) · Pulsadores (7..11) · Buzzer (16, PWM) · SSD1306 (14/15, I2C1)
          │
          ▼
   PicoReflex — Core 0 (juego)  ⇄ estado compartido ⇄  Core 1 (red + display)
          │                                                 │
          ├── ADC3 (GPIO29) → VSYS/3 (batería)              ├── cyw43_arch + lwIP
          └── Flash (récords/ranking/semilla)               └── HTTP + DNS captive portal
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
