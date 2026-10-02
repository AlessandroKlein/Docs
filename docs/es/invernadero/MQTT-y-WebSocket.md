# MQTT y WebSocket

> **Tipo:** API/Red | **Estado:** Estable | **Fecha:** 2026-10-02

## 1. MQTT

Se activa si `mqtt_host` está configurado. Tópicos con prefijo
`greenhouse/{device_id}`:

| Tópico | Dirección | Contenido |
|--------|-----------|-----------|
| `.../state` | ESP32 → broker | Estado general |
| `.../sensors` | ESP32 → broker | Lecturas de sensores |
| `.../actuators` | ESP32 → broker | Estado de actuadores |
| `.../weather` | ESP32 → broker | Estación meteorológica (si `weather.enabled`) |
| `.../cmd` | broker → ESP32 | Comandos (OTA, control) |

Publicación cada 10 s (vía `Scheduler`). El cliente MQTT usa la interfaz activa
(WiFi o Ethernet, según `net_interface`).

### Comando OTA (ejemplo)

```json
{ "type": "ota", "version": "3.15.0", "url": "http://.../firmware.bin", "sha256": "..." }
```

## 2. WebSocket

Endpoint `ws://<IP>:81/ws` para datos en tiempo real:

```json
{ "type": "sensor_update", "sensor": "TEMP-01", "value": 24.7, "unit": "C" }
{ "type": "alarm", "severity": "WARNING", "message": "Sin caudal" }
{ "type": "device_state", "state": "RUN" }
```

## 3. Telemetría (modelo común)

```json
{
  "timestamp": "2026-10-02T12:00:00-03:00",
  "device_id": "GH001",
  "resource_id": "TEMP-01",
  "value": 24.6,
  "unit": "C",
  "quality": "GOOD"
}
```

`quality`: `GOOD | WARNING | INVALID | TIMEOUT | OUT_OF_RANGE | CALIBRATION | DISCONNECTED`.

## 4. Autonomía

Si MQTT/HTTP fallan, el dispositivo sigue funcionando con la última configuración
válida almacenada localmente.
