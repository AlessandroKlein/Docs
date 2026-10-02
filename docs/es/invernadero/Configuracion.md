---
tags:
  - invernadero
  - configuracion
---

# Configuración

> **Tipo:** Configuración | **Estado:** Estable | **Fecha:** 2026-10-02

La configuración se guarda en **NVS** como JSON y se edita por web o `PUT /api/v1/config`.

## Interfaz de red (WiFi o Ethernet)

```json
{
  "network": {
    "net_interface": "wifi",      // "wifi" | "ethernet"
    "eth_cs": 5,                  // CS del W5500
    "eth_dhcp": true,             // true = DHCP; false = IP estática
    "eth_ip": "",
    "eth_gateway": "",
    "eth_mask": "",
    "eth_dns": ""
  }
}
```

El usuario elige con qué interfaz se conecta. Solo una queda activa por arranque.

## Fuente, versionado y reset

- `configuration_source`: `LOCAL` | `CENTRAL`.
- `config_version` se incrementa en cada cambio; se guarda CONFIG ACTUAL/ANTERIOR.
- Reset: `network` / `automation` / `factory` vía `POST /api/v1/reset`.

## Valores por defecto (seguros)

- Temperatura 18/24/28 °C · Humedad 55/70/85 % · Suelo 35/55 %.
- Riego automático deshabilitado hasta configuración.

## 3. Configurabilidad (¿qué se puede cambiar sin tocar código?)

**Configurable desde la web/API (v3.29.0 — sin recompilar):**

- **Pines y direcciones I²C**: `GET/PUT /api/v1/pins` + formulario `/pins` (NVS `ghpins`).
- **Catálogo de sensores**: `PUT /api/v1/sensors/catalog` — `enabled`, `address`,
  `zone`, `bus_index`, `read_interval_ms` por sensor (NVS `ghsensors`).
- **Catálogo de expansores**: `PUT /api/v1/hardware` — agregar/editar nodos
  (HC595/HC165/MCP23017/MCP23S17/ADC) con `enabled`, `address`, `bus_index`
  (NVS `ghhw`).
- Sensores: habilitar/deshabilitar (sht31, ds18b20, suelo, luz, CO₂, lluvia, viento, tanque, caudal, pH, EC, exterior).
- Actuadores: habilitar + cantidad (bomba, válvulas, ventiladores, extractores, luces, ...).
- Umbrales, calibración, zonas, horarios, reglas.
- Red: WiFi/Ethernet, MQTT, NTP, DNS, servidor central.
- Token de API, canal de actualización, simulación, perfiles Modbus.

**Bloqueo de pines (PCB fabricada):**

- `GH_PINS_LOCKED = 1` en `Version.hpp` → `PUT /api/v1/pins` responde **403** y
  `/pins` deshabilita el formulario. El catálogo de sensores/actuadores sigue
  configurable.
- `GH_PINS_LOCKED = 0` (público) → todo editable.

**Todavía requiere recompilar:**

- Agregar un **modelo de sensor nuevo** (los drivers están compilados; el catálogo
  elige entre los soportados).
- Asignar **canales** a actuadores (hoy el `ROLE_TABLE` tiene canales fijos 0..24).

> Detalle y roadmap en `docs/MEJORAS.md` §13 y
> [Referencia de configuración](Referencia-configuracion.md).
