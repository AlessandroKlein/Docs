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

**Configurable desde la web/API:**

- Sensores: habilitar/deshabilitar (sht31, ds18b20, suelo, luz, CO₂, lluvia, viento, tanque, caudal, pH, EC, exterior).
- Actuadores: habilitar + cantidad (bomba, válvulas, ventiladores, extractores, luces, ...).
- Umbrales, calibración, zonas, horarios, reglas.
- Red: WiFi/Ethernet, MQTT, NTP, DNS, servidor central.
- Token de API, canal de actualización, simulación, perfiles Modbus.

**NO configurable (compile-time):**

- Mapa de pines (`PinMap.hpp`).
- Selección del modelo de sensor (drivers compilados).
- Expansores (74HC165/MCP23S17/ADC) y sus pines/CS.
- Direcciones I²C.

> Detalle y roadmap en `docs/MEJORAS.md` §13.
