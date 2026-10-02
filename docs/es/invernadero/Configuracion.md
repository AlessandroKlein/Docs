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
