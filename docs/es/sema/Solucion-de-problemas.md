---
tags:
  - sema
  - soporte
---

# Solución de problemas

> **Tipo:** Soporte | **Estado:** Estable | **Firmware:** v1.22.0

## No conecta a WiFi

1. Verificar SSID/clave en `PUT /api/v1/config` (o dashboard).
2. Comprobar que `network.mode == "STA"`.
3. Si no llega a la red, reiniciar y revisar el monitor serial.

## No aparece `sema-001.local` (mDNS)

- Verificar `network.mdns == true` y que el cliente soporte mDNS.
- Usar la IP directa (`GET /api/v1/network` para verla).

## Sensor no detectado (I²C)

- Revisar `GET /api/v1/diagnostics` → `i2c_devices`.
- Verificar pull-ups de 4,7 kΩ en SDA/SCL.
- Comprobar la dirección I²C correcta.

## Mediciones en `SENSOR_DISCONNECTED` / error

- Verificar cableado y alimentación del sensor.
- `GET /api/v1/health` → `sensors.online` vs `sensors.total`.

## `401 unauthorized`

- Enviar `X-API-Key: <api_key>` (o `server_key`).
- Si ambas claves están vacías, el acceso está abierto (primera config).

## Login bloqueado (`429`)

- Esperar 60 s (rate limit de 5 intentos).

## Sesión expira

- La sesión dura 1 h sin actividad; volver a iniciar sesión.

## Reboot inesperado

- `GET /api/v1/diagnostics` → `reset_reason` para identificar la causa (watchdog,
  excepción, etc.).

## OTA falla

- Verificar `X-API-Key` y que el binario sea el de la board correcta.
- Revisar espacio libre en flash (`pio run` reporta el uso).

## Monitor serial

```bash
pio device monitor -b 115200
```
