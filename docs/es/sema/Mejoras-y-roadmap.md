---
tags:
  - sema
  - roadmap
---

# Mejoras y roadmap

> **Tipo:** Roadmap | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Estado del desarrollo y de la **Definition of Done** (`README.md` §103), verificado
contra el código. Estado de cada ítem: ✅ hecho · 🔄 parcial · ⬜ pendiente.

## 1. Definition of Done

| Requisito (§103) | Estado | Nota |
|------------------|:------:|------|
| Core funcionando | ✅ | `SemaCore`, registry, event bus, scheduler |
| Configuración persistente | ✅ | NVS, schema versionado, transaccional |
| Web local | ✅ | Dashboard + páginas `/config/*`, `/sensors`, `/events` |
| API | ✅ | 53 rutas HTTP registradas (`HttpServer::begin`) |
| Dashboard modular | ✅ | Tarjetas Gridstack, histórico, gráficos, edición |
| Sistema de módulos | 🔄 | Interfaz + registro; sin módulos instalables/desinstalables |
| Sistema de sensores | ✅ | `Sensor`/`SensorManager`/`SensorFactory` |
| Catálogo de sensores | ✅ | 17 modelos configurables vía `sensors[]` |
| Detección I²C | ✅ | `I2cScanner` → `/api/v1/diagnostics` |
| Detección 1-Wire | ✅ | Multi-dispositivo DS18B20 en el bus |
| Configuración GPIO | ✅ | `gpio[]` + `/api/v1/gpio` GET/POST |
| Configuración ADC | ✅ | `ADC`, `ADS1115` como sensores configurables |
| MCP23017 / 74HC595 / 74HC165 | ✅ | v1.14.0 / v1.17.0 |
| RS485 / Modbus RTU / CAN | ✅ | v1.20.0 / v1.21.0 |
| LoRa / Zigbee | ✅ | v1.22.0 / v1.23.0 |
| Ethernet | ✅ | LAN8720A (RMII) y W5500 (SPI), v1.24–v1.25 |
| Medición energética | 🔄 | Batería por ADC; falta corriente/consumo |
| Almacenamiento | ✅ | NVS (config) + LittleFS (eventos) + microSD (histórico; sin SD no hay histórico) |
| Histórico | ✅ | JSONL, rotación, retención, agregados por hora, CSV |
| Alarmas | ✅ | `RuleEngine` + `EventLog` persistente |
| Diagnóstico | 🔄 | Health monitor + endpoints; falta detalle por sensor |
| Calibración | ✅ | `calibrations[]` por canal |
| Validación de configuración | ✅ | Rechazo transaccional con rollback |
| Backup / importación / exportación | ✅ | `/api/v1/backup` GET/POST |
| OTA | ✅ | Particiones A/B, rollback, verificación SHA-256 |
| Seguridad | 🔄 | `api_key` + `server_key` + login + rate limit + sesión; falta TLS/RBAC |
| Watchdog | ✅ | `Watchdog` con heartbeat del core |
| Documentación | ✅ | Este apartado SEMA + repo de código |

---

## 2. Pendientes reales

### 2.1 Parciales (requieren más trabajo)

| Ítem | Estado actual | Qué falta |
|------|---------------|-----------|
| Sistema de módulos | Interfaz + registro (`Module.hpp`, `ModuleRegistry`) | Módulos reales con instalación/desinstalación/permisos |
| Medición energética | Tensión de batería por ADC | Corriente, potencia y consumo |
| Diagnóstico | `HealthMonitor` + `/api/v1/diagnostics` | Diagnóstico por sensor más profundo (histéresis de fallos, contadores) |
| Seguridad | `api_key` + `server_key` + login + rate limiting + `extra_keys` revocables | TLS/HTTPS, RBAC completo, OTA firmado |
| Comunicaciones remotas | MQTT, LoRa, Zigbee, Ethernet | Servidor Central (fuera de alcance) |
| Energía | Batería, perfiles, deep sleep | Gestión de carga/MPPT del panel solar |

### 2.2 Fuera de alcance

| Ítem | Motivo |
|------|--------|
| Servidor Central (multiestación, mapas, alertas) | Es un proyecto separado que agrega varios proyectos; SEMA solo se conecta con `server_key` (`SEMA_PROTOCOL_VERSION = 1`) |

### 2.3 Futuras / opcionales

El catálogo completo está en [Futuro](Futuro.md). Resumen por área:

| Área | Ítems principales |
|------|-------------------|
| Seguridad | TLS/HTTPS, OTA firmado, RBAC y rotación de claves |
| Comunicaciones | LoRaWAN, red Zigbee multi-dispositivo, BLE, CoAP/MQTT-SN |
| Energía | MPPT/carga solar, curvas LiFePO4/Li-ion, perfiles por horario, wake por RTC |
| Fiabilidad | RTC hardware (DS3231), config fail-safe, watchdog jerárquico |
| Plataforma | Multi-estación en un dashboard, zona horaria en timestamps |

---

## 3. Próximos pasos ordenados

| Prioridad | Ítem | Tipo | Motivo |
|:---------:|------|------|--------|
| 1 | **TLS/HTTPS** para la web y la API | Software | Hoy todo el tráfico local va en claro |
| 2 | **OTA firmado** | Software | Verifica integridad (SHA-256) pero no autenticidad |
| 3 | RBAC y rotación de claves | Software | `extra_keys` cubre revocación, no roles |
| 4 | Módulos reales (instalar/desinstalar) | Software | Completa el ciclo de vida prometido en la arquitectura |
| 5 | Medición de corriente/consumo | Hardware | Necesita sensor (INA219/ACS712) |
| 6 | Gestión MPPT / carga solar | Hardware | Requiere controlador de carga compatible |
| 7 | RTC DS3231 | Hardware | Hora válida sin NTP y wake por alarma |
| 8 | Multi-estación (Servidor Central) | Proyecto aparte | Fuera del alcance de SEMA |

---

## 4. Regla de actualización

Este archivo se actualiza en el mismo ciclo que el código: cada ítem que cambia de
estado se refleja aquí, en el [CHANGELOG](CHANGELOG.md) y en la
[Evolución](Evolucion.md). Es el equivalente en el repo Docs de
`docs/MEJORAS.md` y `docs/PENDIENTES.md` del repo de código.

---

## Ver también

- [Evolución](Evolucion.md) · [Futuro](Futuro.md) · [CHANGELOG](CHANGELOG.md)
- [Registro de cambios](Registro-de-cambios.md) · [Estadísticas y métricas](Estadisticas-y-metricas.md)
