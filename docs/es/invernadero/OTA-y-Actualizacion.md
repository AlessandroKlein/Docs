# OTA y actualización

> **Tipo:** Embebidos | **Estado:** Estable | **Fecha:** 2026-10-02

## 1. Actualización OTA

- ArduinoOTA (hostname configurado, por defecto `invernadero`).
- Doble partición (app0/app1) → rollback automático si el firmware nuevo no arranca.
- Al finalizar se confirma (`esp_ota_mark_app_valid_cancel_rollback`).

## 2. Tabla de particiones

| Archivo | Flash | app0/app1 | SPIFFS |
|---------|-------|-----------|--------|
| `default.csv` | 4 MB | 1,25 MB c/u | 1,375 MB |
| `default_8MB.csv` | 8 MB | 3,19 MB c/u | 1,5 MB |
| `default_16MB.csv` | 16 MB | 6,25 MB c/u | 3,375 MB |

Por defecto: `default_8MB.csv` (8 MB).

## 3. Manifest de firmware

`firmware_manifest.json` (alojado en GitHub):

```json
{
  "project": "Invernadero",
  "channel": "stable",
  "version": "3.15.0",
  "firmware_url": "https://.../firmware.bin",
  "sha256": "...",
  "hardware_profile": "ESP32-GH-V1",
  "config_schema": 2
}
```

## 4. Canales

`stable` (default), `beta`, `development`.

## 5. Proceso de rollback

```text
Descarga → Verificación SHA-256 → Instalación → Reinicio →
Boot de prueba → Health Check → OK=CONFIRMAR / ERROR=ROLLBACK
```

## 6. OTA desde el servidor central

1. Subir `.bin` (`POST /api/v1/firmware/upload`), calcula SHA-256.
2. Actualizar (`POST /api/v1/devices/{id}/ota`) → comando MQTT.
3. El ESP32 descarga, verifica SHA-256 e instala sin perder NVS ni SPIFFS.

> Nota: OTA por **Ethernet (W5500)** está bloqueado (HTTPClient solo acepta
> WiFiClient) — ver `docs/MEJORAS.md` §10.
