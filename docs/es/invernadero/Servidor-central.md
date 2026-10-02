---
tags:
  - servidor
---

# Servidor central (multi-proyecto)

> **Tipo:** Backend/Web | **Estado:** Especificación | **Fecha:** 2026-10-02

Documento de diseño del servidor central unificado para **todos los proyectos**.
Detalle completo en [`docs/SERVIDOR-CENTRAL.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/SERVIDOR-CENTRAL.md).

## Stack actual (rama `server`)

| Capa | Tecnología |
|------|-----------|
| Frontend | HTML + CSS + JS (Chart.js, mqtt.js) |
| Backend / API | PHP 8.2 + Apache (front controller) |
| Base de datos | PostgreSQL 16 |
| Mensajería | Mosquitto (MQTT 1883 + WebSocket 9001) |
| Orquestación | Docker Compose |

## API (resumen)

`POST /api/v1/auth/login` (JWT) · `GET/POST /api/v1/greenhouses` ·
`GET /api/v1/devices` · `GET /api/v1/devices/{id}` ·
`.../{id}/sensors|actuators|alarms|readings|shadow|config|ota` ·
`POST /api/v1/firmware/upload`.

## Seguridad

JWT · contraseñas bcrypt · RBAC granular · token de API por dispositivo ·
**rate limiting por IP** · **CORS configurable** · MQTT con TLS en producción.

## Store & Forward

Columna `sequence` en `sensor_readings` + consulta con `after_sequence` para
sincronización incremental (detección de huecos).

## Modelo multi-proyecto

```text
projects → devices → sensors/actuators → readings/states/events
```

Aislamiento por `project_id` + scope JWT + prefijo MQTT `{project}/{device_id}/...`.

## Puesta en marcha

```bash
cd server
cp .env.example .env
docker compose up -d --build
docker compose exec php php mqtt/worker.php
```
