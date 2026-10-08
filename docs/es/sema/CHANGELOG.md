---
tags:
  - sema
  - changelog
---

# Registro de versiones (CHANGELOG)

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.103.0

Historial de versiones de SEMA (resumen). El changelog completo con enlaces está en
[`CHANGELOG.md`](https://github.com/AlessandroKlein/SEMA/blob/main/CHANGELOG.md).

## 1.x (estable)

| Versión | Resumen |
|---------|---------|
| **1.103.0** | MAX14830 4 puertos UART |
| **1.102.0** | Eliminar data/ y LittleFS web |
| **1.101.0** | Bus I2C/UART por expansores |
| **1.100.0** | Gridstack PROGMEM gzip |
| **1.99.0** | Expansores en Pines de buses |
| **1.98.0** | Expansores SPI MAX14830/SC18IS602B |
| **1.97.0** | Reservados visibles + SPI WROOM |
| **1.96.0** | I2C guardado propio + reinicio |
| **1.95.0** | SPI + RMII reservados |
| **1.94.0** | Pines de buses editables |
| **1.93.0** | Reserva pines MCP |
| **1.92.0** | General, SD CS, buses, demo MCP |
| **1.91.0** | MCP23S17 expansor |
| **1.90.0** | Modales personalizados |
| **1.89.0** | Pines reservados + defaults |
| **1.88.0** | CSS pulido + pines deshabilitados |
| **1.87.0** | Quitar pie tarjeta |
| **1.86.0** | Tarjetas espanol + traducciones |
| **1.85.0** | WiFi y mDNS separados |
| **1.84.0** | Fix datos WiFi |
| **1.83.0** | Temp ESP + mDNS + estilos |
| **1.82.0** | UI WiFi + bus I2C unico |
| **1.81.0** | Derivadas demo + datos ESP |
| **1.80.0** | Selectores de pines GPIO |
| **1.79.0** | Demo mas completa |
| **1.78.0** | Fix conversion unidades demo |
| **1.77.0** | Nav unificado + titulos |
| **1.76.0** | Fix guardado NVS |
| **1.75.0** | Multi-idioma es/en |
| **1.74.0** | Fix tema config |
| **1.73.0** | Selector de unidades |
| **1.72.0** | Paginas por seccion |
| **1.71.0** | Quitar seccion GPIO |
| **1.70.0** | Agregacion por niveles |
| **1.69.0** | Shift register beta (flag) |
| **1.68.0** | microSD configurable |
| **1.67.0** | Watchdog jerarquico |
| **1.66.0** | Pines por board + SD SPI |
| **1.65.0** | Zona horaria + fail-safe |
| **1.64.0** | Registros cascada + tooltips |
| **1.63.0** | Pines RMII Ethernet |
| **1.62.0** | MCP23S17 + registro SPI |
| **1.61.0** | Expansores separados |
| **1.60.0** | Canal ADS1115 por sensor |
| **1.59.0** | ADS1115 canal + magnitud |
| **1.58.0** | DS18B20 por ROM |
| **1.57.0** | DS18B20 un pin + ROM |
| **1.56.0** | Fuente ADC + DS18B20 |
| **1.55.0** | Sensores especificos |
| **1.54.0** | Tooltips sensores |
| **1.53.0** | Switches + expansores |
| **1.52.0** | Sensores seguros + I2C |
| **1.51.0** | Config sensores + pines |
| **1.50.0** | Dashboard publico + footer |
| **1.49.0** | Layout + modo edicion |
| **1.48.0** | Fix tarjetas + layout |
| **1.47.0** | Fix dashboard + OTA |
| **1.46.0** | Buffer JSON + pines |
| **1.45.0** | Fix guardar + reloj |
| **1.44.0** | Demo + sensores + mDNS |
| **1.43.0** | CSS unificado + mDNS |
| **1.42.0** | Fix auth config + NTP |
| **1.41.0** | Rutas /config + NTP + OTA |
| **1.40.0** | Multi-pagina + red + claves |
| **1.39.0** | Fix parpadeo + escaneo |
| **1.38.0** | Fix Gridstack |
| **1.37.0** | Escaneo WiFi + login |
| **1.36.0** | Fix guardar + red separada |
| **1.35.0** | OTA checksum |
| **1.34.0** | Tema claro/oscuro |
| **1.33.0** | Export CSV + retención |
| **1.32.0** | Gráficos avanzados + doc |
| **1.31.0** | Tarjetas añadibles + gráficos |
| **1.30.0** | Gridstack offline (LittleFS) |
| **1.29.0** | Gridstack + veleta WH-SP-WD |
| **1.28.0** | Unidades + magnitudes derivadas |
| **1.27.0** | Binarios por board + particiones 4/8/16 MB |
| **1.26.0** | Perfil de hardware (build_flags) |
| **1.25.0** | W5500 en lwIP (ESP-IDF) |
| **1.24.0** | Ethernet (LAN8720A + W5500) |
| **1.23.0** | Zigbee (CC2652P2) |
| **1.22.0** | LoRa (SX1262) |
| **1.21.0** | CAN 2.0 (TWAI) |
| **1.20.0** | RS485/Modbus RTU |
| **1.19.0** | Sensor SOLAR (radiación solar) |
| **1.18.0** | Sensor CO (monóxido de carbono) |
| **1.17.0** | Shift registers 74HC595/74HC165 |
| **1.16.0** | Cookie de sesión `SameSite=Strict` (mitiga CSRF) |
| **1.15.0** | Sensor ADS1115 (ADC externo 16 bits, I²C) |
| **1.14.0** | Expansor MCP23017 (16 GPIO por I²C) |
| **1.13.0** | Sensor AS3935 (detección de rayos) |
| **1.12.0** | Expiración de sesión del login (1 h) |
| **1.11.0** | Autenticación MQTT (usuario/contraseña) |
| **1.10.0** | Sensor SGP30 (eCO₂/TVOC) |
| **1.9.0** | Hot reload de sensores |
| **1.8.0** | Hot reload de publicadores |
| **1.7.0** | GPIO standalone configurable + API `/api/v1/gpio` |
| **1.6.0** | Rate limiting en el login |
| **1.5.0** | Edición de config desde la UI |
| **1.4.0** | Gráfico de temperatura en el dashboard |
| **1.3.0** | Histórico en el dashboard |
| **1.2.0** | Hot reload de reglas/calibración |
| **1.1.0** | Rotación del EventLog |
| **1.0.0** | Primera versión estable |

## 0.x (desarrollo)

| Versión | Resumen |
|---------|---------|
| 0.56.0 | Login web con sesión |
| 0.55.0 | NTP/RTC |
| 0.54.0 | Board Profile (pines fijos) |
| 0.53.0 | Rotación del histórico |
| 0.52.0 y anteriores | Core, sensores, API, OTA, energía, watchdog, health, eventos/alarmas |
