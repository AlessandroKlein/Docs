---
tags:
  - invernadero
  - soporte
---

# Solución de problemas

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-02

Diagnóstico de los problemas más frecuentes, con el endpoint o comando que lo
confirma.

## 1. Herramientas de diagnóstico

| Endpoint | Qué muestra |
|----------|-------------|
| `GET /api/v1/status` | Estado general, versión, uptime |
| `GET /api/v1/diagnostics` | WiFi/RSSI, MQTT, heap, uptime |
| `GET /api/v1/health` | Heartbeat por tarea, stack HWM, heap |
| `GET /api/v1/boot` | Contadores de reinicio (watchdog, brownout, panic) |
| `GET /api/v1/logs` | Log estructurado (nivel, módulo, mensaje) |
| `GET /api/v1/detect` | Escaneo I²C → tipo de dispositivo por dirección |
| `GET /api/v1/rs485` | Estadísticas Modbus (tx, rx, CRC, timeouts) |
| `GET /api/v1/devices` / `/sensors` | Valores y calidad por sensor |

## 2. No arranca o reinicia en bucle

1. `GET /api/v1/boot` → mirar `watchdog_count`, `brownout_count`, `panic_count`.
2. **Brownout** → fuente insuficiente. Usar 5 V / 2 A y capacitor de desacople.
3. **Panic** → leer `/api/v1/logs` (suele ser un pin inválido o conflicto).
4. **Watchdog** → una tarea quedó bloqueada (revisar sensor I²C que no responde;
   los drivers usan timeout).
5. Verificar que GPIO 12/15 (strapping) no estén en nivel incorrecto al arrancar.

## 3. No conecta a WiFi

1. `GET /api/v1/diagnostics` → verificar que `ssid` no esté vacío.
2. Recordar: **ESP32 solo soporta WiFi 2,4 GHz** (no 5 GHz).
3. El AP de configuración (`Invernadero-AP`) aparece si no hay credenciales.
4. `POST /api/v1/network/scan` lista las redes visibles.

## 4. No responde `invernadero.local`

1. mDNS puede estar bloqueado en la red → usar la **IP** (`GET /api/v1/status`).
2. En Windows puede requerir Bonjour; en Linux `avahi`.
3. Verificar que estés en la **misma red/subred**.

## 5. Un sensor marca `DISCONNECTED`

1. `GET /api/v1/detect` → ¿aparece la dirección I²C?
2. Verificar **pull-ups 4,7 kΩ** en SDA/SCL.
3. Verificar alimentación 3,3 V del sensor.
4. Revisar que la dirección configurada coincida (`/pins` → `i2c_addr_*`).

## 6. Un sensor marca `ERROR` (presente pero falla)

1. Hardware detectado pero lectura inválida → revisar cableado.
2. Para DS18B20: verificar la resistencia de 4,7 kΩ en el pin 1-Wire.
3. Para suelo: revisar calibración (`soil_dry` / `soil_wet`).

## 7. HR/CRC en RS485/Modbus

1. `GET /api/v1/rs485` → ver `crcErrors`, `timeouts`.
2. Verificar **baudios** (`rs485_baud`) coincidentes con el esclavo.
3. Verificar **paridad** y bits de stop.
4. **Terminación 120 Ω** en los extremos del bus.
5. `POST /api/v1/rs485/scan` para descubrir esclavos.

## 8. Un actuador no responde

1. `GET /api/v1/actuators` → ver `enabled` y `output`.
2. ¿El rol está habilitado en `actuators` del config?
3. Para 74HC595: verificar `hc595_mosi/sclk/latch` y `hc595_count`.
4. Para canal ≥ 32 (MCP23017): verificar que el nodo MCP23017 esté **habilitado**
   en `GET /api/v1/hardware`.
5. Jerarquía: una sobrescritura de **seguridad** (emergencia) fuerza 0.

## 9. No puedo guardar los pines (403)

- `GH_PINS_LOCKED = 1` → la PCB es fija y los pines están bloqueados por diseño.
- Para editarlos, recompilar con `#define GH_PINS_LOCKED 0`.

## 10. OTA falla

1. Verificar **SHA-256** del `.bin` (si no coincide, se aborta por seguridad).
2. Con Ethernet: solo **HTTP** (HTTPS requiere TLS, no disponible en W5500 clásico).
3. Verificar partición de 8 MB (`default_8MB.csv`) y espacio libre.
4. Ver `GET /api/v1/ota` y los logs.

## 11. La configuración no se guarda

1. `PUT /api/v1/config` requiere **autenticación**.
2. Verificar `schema_version` (debe migrar a 2).
3. Usar `POST /api/v1/config/rollback` si quedó en mal estado.
4. Último recurso: `POST /api/v1/factory-reset`.

## 12. MQTT no conecta

1. `GET /api/v1/diagnostics` → `mqtt_host` no vacío.
2. Verificar usuario/clave y que el broker sea accesible.
3. Los tópicos usan el prefijo `greenhouse/{device_id}`.

## 13. Códigos de calidad de datos

| Calidad | Significado | Acción |
|---------|-------------|--------|
| `GOOD` | Lectura válida | — |
| `WARNING` | Dudosa | Vigilar |
| `INVALID` | Valor imposible | Revisar calibración |
| `TIMEOUT` | Sin respuesta | Revisar bus |
| `OUT_OF_RANGE` | Fuera de límites | Ajustar umbrales |
| `DISCONNECTED` | No detectado | Revisar cableado/alimentación |

Ver también: [Identidad y estados](Identidad-y-Estados.md) · [FAQ](FAQ.md).
