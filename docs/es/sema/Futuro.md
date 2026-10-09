---
tags:
  - sema
  - futuro
---

# Mejoras opcionales y futuras

> **Tipo:** Catálogo de ideas opcionales | **Estado:** Catálogo vivo | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Registro de ítems **opcionales**: no forman parte de la Definition of Done (§103) ni
bloquean un release. Cada ítem es una idea, no un compromiso. Los ítems que ya se
implementaron se retiran del catálogo y pasan al [CHANGELOG](CHANGELOG.md).

> ⚠️ **Corrección respecto de `docs/FUTURO.md` del repo de código**: la versión del
> 2026-10-03 daba por pendientes varios ítems que **ya están en el firmware**
> (rotación del `EventLog`, agregación del histórico, export CSV, retención por tiempo,
> tema claro/oscuro, rate limiting, expiración de sesión, gráficos, edición de config
> desde la UI, páginas por sección, CAN, zona horaria y soporte de microSD). Se listan
> abajo en §10 para dejar constancia.

## 1. Seguridad

| Ítem | Nota | Estado |
|------|------|:------:|
| TLS/HTTPS (certificado) | Cifrar la web local y la API; hoy es HTTP en claro | ⬜ |
| OTA con firma | Verifica integridad (SHA-256) pero no autenticidad del binario | ⬜ |
| RBAC (roles/permisos) | Hoy `api_key`/`server_key`/`extra_keys` no tienen roles | ⬜ |
| Rotación de claves asistida | `extra_keys` permite revocar, pero no hay rotación guiada | ⬜ |
| Bloqueo por IP tras N intentos | Hoy el rate limit del login es global | ⬜ |

## 2. Dashboard y web

| Ítem | Nota | Estado |
|------|------|:------:|
| Vistas guardadas del dashboard | Hoy el layout Gridstack es global, no por usuario | ⬜ |
| Exportar imágenes/PDF del panel | Reporte visual de la estación | ⬜ |
| Modo kiosco / pantalla completa | Panel permanente en un display | ⬜ |
| Accesibilidad (contraste, teclado) | Cumplimiento WCAG básico | ⬜ |

## 3. Almacenamiento y datos

| Ítem | Nota | Estado |
|------|------|:------:|
| Retención por niveles configurable | La agregación horaria existe; falta hacerla configurable | 🔄 |
| Exportación JSON masiva | Hoy `GET /api/v1/history` devuelve JSON; el CSV ya existe | 🔄 |
| Descarga de eventos en CSV | Igual que el histórico, para `events`/`alarms` | ⬜ |
| Réplica a servidor (store & forward) | Enviar histórico pendiente al recuperar enlace | ⬜ |

## 4. Comunicaciones

| Ítem | Nota | Estado |
|------|------|:------:|
| LoRaWAN | Protocolo de red sobre LoRa (hoy LoRa punto a punto) | ⬜ |
| Red Zigbee multi-dispositivo | Sensores/actuadores Zigbee remotos (hoy ZNP como co-procesador) | ⬜ |
| BLE | Configuración inicial por Bluetooth | ⬜ |
| CoAP / MQTT-SN | Protocolos ligeros para redes de muy bajo consumo | ⬜ |
| Ethernet con PoE | Alimentación por el mismo cable | ⬜ |

## 5. Sensores adicionales

| Ítem | Nota | Estado |
|------|------|:------:|
| PT100/PT1000/NTC | Temperatura industrial con acondicionamiento | ⬜ |
| Sensores Modbus adicionales | pH, EC, NPK, humedad de suelo industrial | 🔄 |
| Anemómetro ultrasónico | Sin partes móviles | ⬜ |
| Piranómetro con calibración multipunto | Hoy `SOLAR` usa escala/offset | 🔄 |
| PMSA003 / SPS30 | Alternativas al PMS5003 por UART | ⬜ |

## 6. Energía

| Ítem | Nota | Estado |
|------|------|:------:|
| Gestión de carga / MPPT | Voltaje y corriente de carga del panel solar | ⬜ |
| Medición de corriente y consumo | Requiere sensor (INA219/ACS712) | ⬜ |
| Batería LiFePO4 / Li-ion | Curvas de descarga para estimar estado de carga | ⬜ |
| Perfiles energéticos por horario | Ahorro programado (hoy perfiles por estado) | ⬜ |
| Wake por RTC (alarmas de hora) | Despertar a horas fijas, no solo por timer/lluvia | ⬜ |

## 7. Hardware / expansión

| Ítem | Nota | Estado |
|------|------|:------:|
| PCB propia (rev1) | Hoy los pines fijos existen para la placa `esp32-wroom-32u` en diseño | 🔄 |
| Driver propio de expansores SPI | MAX14830/SC18IS602B hoy son configuración de bus | 🔄 |
| Salidas de potencia integradas | Relés/SSR en placa, con protecciones | ⬜ |

## 8. Fiabilidad y operación

| Ítem | Nota | Estado |
|------|------|:------:|
| RTC hardware (DS3231) con batería | Hora válida sin NTP | ⬜ |
| Config fail-safe con copia previa | Hoy, si la config no valida, se cargan los defaults | 🔄 |
| Watchdog jerárquico por tarea | Un watchdog por tarea crítica | ⬜ |
| Rollback automático por arranques fallidos | Contador de arranques fallidos → partición anterior | ⬜ |
| Log remoto (syslog/MQTT) | Diagnóstico sin acceso físico | ⬜ |

## 9. Plataforma

| Ítem | Nota | Estado |
|------|------|:------:|
| Multi-estación en un dashboard | Requiere el Servidor Central (fuera de alcance) | ⬜ |
| Alertas push (Telegram/webhook) | Hoy los eventos se publican por MQTT/HTTP | 🔄 |
| OTA masivo por flota | Actualizar varias estaciones a la vez | ⬜ |
| App móvil | Acceso rápido a estado y alarmas | ⬜ |

## 10. Ya implementado (retirado del catálogo)

| Ítem | Dónde se implementó |
|------|---------------------|
| Rotación del `EventLog` (D-0057) | `EventLog::rotate()` (conserva las últimas `maxEntries`) |
| Agregación del histórico por niveles | `HistoryStore::aggregate()` + `readAggregated()`, invocada desde `SemaCore` |
| Retención del histórico por tiempo | `storage.retention_days` → `HistoryStore::setRetentionSeconds()` |
| Export CSV del histórico | `GET /api/v1/history?format=csv` (v1.33.0) |
| Gráficos con series, ejes y rangos | v1.32.0 |
| Edición de configuración desde la UI | Páginas `/config/*` + formularios del dashboard |
| Páginas por sección | `/sensors`, `/events`, `/config/network`, `/config/security`, `/config/system`, `/config/wind`, `/config/sensors` |
| Tema claro/oscuro persistente | v1.34.0 |
| Rate limiting y expiración de sesión | v1.6.0 / v1.12.0 |
| CAN (TWAI) | v1.21.0 |
| Zona horaria configurable | `system.timezone` (default `America/Argentina/Buenos_Aires`) |
| Soporte de microSD | `storage.sd_enabled` + `HistoryStore::enableSd()` |
| Health por sensor | Campo `healthy` en el catálogo de `/api/v1/sensors` |
| Dashboard multi-idioma (es/en) | `system.lang` + `data-i18n` con `toggleLang()` en `HttpServer.cpp` |
| Layout del dashboard persistente | `system.dashboard_layout` + `/api/v1/dashboard/layout` |

---

## 11. Regla de actualización

Este archivo se actualiza al **retirar** un ítem (cuando se implementa y pasa al
[CHANGELOG](CHANGELOG.md) y a [Evolución](Evolucion.md)) o al **agregar** ideas nuevas.
Es docs-only: no requiere release.

---

## Ver también

- [Mejoras y roadmap](Mejoras-y-roadmap.md) · [Evolución](Evolucion.md) · [CHANGELOG](CHANGELOG.md)
- [Seguridad](Seguridad.md) · [Energía y consumo](Energia-y-consumo.md) · [Almacenamiento e histórico](Almacenamiento-e-historico.md)
