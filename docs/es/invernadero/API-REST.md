---
tags:
  - invernadero
  - api
---

# API REST (referencia completa)

> **Tipo:** API | **Estado:** Estable | **Fecha:** 2026-10-02

Base: `http://<IP>/api/v1/` (puerto 80). **48 endpoints** en v3.29.0.

!!! info "Autenticación"
    Los endpoints de **escritura** requieren `Authorization: Bearer <token>` o
    `X-Auth-Token` (sesión de admin o token de API). Los de lectura son abiertos.
    Si no hay autorización, el endpoint responde `401`; si los pines están
    bloqueados, `403`.

## Estado y lectura

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/status` | Estado general (temperatura, humedad, suelo, tanque, luz, bomba, ventilador) |
| GET | `/sensors` | Lista de sensores con calidad |
| GET | `/actuators` | Lista de actuadores (output, fault) |
| GET | `/zones` | Zonas configuradas |
| GET | `/weather` | Estación meteorológica externa |
| GET | `/alarms` | Alarmas (severidad ≥ 2) |
| GET | `/events` | Últimos eventos |
| GET | `/logs` | Logs estructurados (nivel, módulo, mensaje) |

## Configuración

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET/PUT | `/config` | Configuración JSON completa (PUT protegido) |
| GET | `/config/schema` | Esquema descriptivo |
| GET | `/config/export` | Exportar configuración |
| POST | `/config/import` | Importar configuración |
| POST | `/config/rollback` | Restaura la configuración anterior |
| POST | `/reset` | `{"level":"network"\|"automation"\|"factory"}` |
| POST | `/factory-reset` | Reset de fábrica completo |

## Control de actuadores

```http
POST /api/v1/actuators
{"role":"pump","index":0,"output":100}
```

Roles: `pump`, `valve`, `fan`, `extractor`, `heater`, `humidifier`, `light`,
`roof`, `window`, `shade`, `alarm`. `output` en % (0..100).

## Automatización (reglas)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/automation` | Lista de reglas |
| POST | `/automation` | Agrega una regla |
| DELETE | `/automation` | Borra reglas |

## Modularidad runtime (v3.18 → v3.29)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET/PUT | `/pins` | Mapa de pines + direcciones I²C (NVS `ghpins`). PUT → **403** si `GH_PINS_LOCKED=1` |
| GET/PUT | `/sensors/catalog` | Catálogo de sensores: `enabled`, `address`, `zone`, `bus_index`, `read_interval_ms` (NVS `ghsensors`) |
| GET/PUT | `/hardware` | Catálogo de expansores: agregar/editar nodos HC595/HC165/MCP23017/MCP23S17/ADC (NVS `ghhw`) |
| GET | `/buses` | Buses registrados (I²C/SPI/UART/RS485/1-Wire/GPIO/CAN) |
| GET | `/detect` | Autodetección I²C: escanea y mapea dirección → tipo de dispositivo |
| GET | `/actuators/catalog` | Catálogo de actuadores |
| GET | `/modules` | Módulos embebidos |
| GET | `/storage` | Almacenamiento (LittleFS/SPIFFS/SD): backend, uso, archivos |

**Ejemplo — leer y escribir pines:**

```bash
curl http://192.168.1.50/api/v1/pins
# {"locked":0,"pins":{"i2c_sda":21,...}}

curl -X PUT http://192.168.1.50/api/v1/pins \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{"i2c_sda":21,"i2c_scl":22,"onewire":4}'
# {"ok":true,"restart":true}
```

## RS485 / Modbus

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/rs485` | Estadísticas del bus (tx, rx, CRC, timeouts) |
| POST | `/rs485/scan` | Escaneo de esclavos (IDs 1..64) |
| GET | `/modbus?slave=1&func=3&reg=0` | Lectura puntual de registros |
| GET | `/modbus/profiles` | Perfiles e instancias Modbus |
| GET | `/modbus/gateway` | Gateway RS485: valores y estado por esclavo |

## Red

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/network` | SSID, IP, interfaz (wifi/ethernet), MQTT, NTP, DNS |
| POST | `/network/scan` | Escaneo de redes WiFi |

## Identidad y diagnóstico

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/device` | UID, id, perfil, firmware, estado, causa de reinicio |
| GET | `/capabilities` | Capacidades (`WIFI`, `I2C`, `SPI`, `RS485`, `MODBUS`, …) |
| GET | `/diagnostics` | WiFi, RSSI, MQTT, heap, uptime |
| GET | `/health` | Health monitor por tareas (stack HWM, heartbeat) |
| GET | `/boot` | Contadores de reinicio (boot/watchdog/brownout/panic/ota) |

## OTA

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/ota` | Estado de la última actualización |
| GET | `/firmware` | Manifest/versión de firmware |

## Seguridad

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/auth/login` | Login de administrador (`{"user","pass"}`) |
| POST | `/token/rotate` | Genera token de API (se muestra una vez) |
| POST | `/token/revoke` | Revoca el token |
| GET | `/token/status` | Estado del token (sin revelarlo) |

## Páginas web (HTML)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/` | Dashboard local |
| GET | `/pins` | Formulario de configuración de pines (se deshabilita si `GH_PINS_LOCKED=1`) |

Ver también: [MQTT y WebSocket](MQTT-y-WebSocket.md) ·
[Seguridad](Seguridad.md) · [Solución de problemas](Solucion-de-problemas.md).
