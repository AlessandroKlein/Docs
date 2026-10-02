---
tags:
  - invernadero
  - ota
---

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
  "version": "3.29.0",
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

## 7. OTA por Ethernet (W5500) — resuelto en v3.16.0

`Update.h` es **agnóstico al origen del stream**: recibe bytes por
`Update.write()` desde cualquier `Client`. Por eso `OtaManager::applyFromUrl()`
hace un **GET HTTP manual** sobre el `Client*` activo (WiFi **o** `EthernetClient`)
y escribe el binario en la partición OTA.

| Caso | Implementación | Estado |
|------|----------------|--------|
| HTTP sobre WiFi | GET manual sobre `Client*` | ✅ |
| HTTP sobre Ethernet (W5500) | GET manual sobre `Client*` | ✅ v3.16.0 |
| HTTPS sobre WiFi | `HTTPClient` + `WiFiClientSecure` | ✅ |
| HTTPS sobre Ethernet | requiere TLS (W5500lwIP / ETH nativo) | ❌ pendiente |

Soporta `Content-Length` y `Transfer-Encoding: chunked` (necesario para servidores
como GitHub/S3).

> El bloqueo histórico era solo de `HTTPClient` (acepta únicamente `WiFiClient`),
> no de `Update`. Ver `docs/MEJORAS.md` §10.
