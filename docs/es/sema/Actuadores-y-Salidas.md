---
tags:
  - sema
  - actuadores
  - gpio
---

# Actuadores y salidas

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.51.0

SEMA controla **salidas digitales** (relés, leds, válvulas, sirenas) a través del
subsistema de **GPIO standalone** (`GpioManager`), con soporte de expansión I²C
(MCP23017).

## Definir una salida (config)

```json
{ "gpio": [
  { "id": "relay_riego", "pin": 26, "mode": "output", "initial": 0, "expander_addr": 0 },
  { "id": "valvula_2",   "pin": 3,  "mode": "output", "initial": 0, "expander_addr": 32 }
] }
```

| Clave | Descripción |
|-------|-------------|
| `id` | Nombre lógico |
| `pin` | GPIO nativo, o pin 0–15 del MCP23017 |
| `mode` | `output` (actuador) o `input*` (lectura) |
| `initial` | Estado al arrancar (`0`/`1`) |
| `expander_addr` | `0` = nativo; `!= 0` = MCP23017 en esa dirección I²C (p. ej. `32` = `0x20`) |

## Escribir una salida (API)

```http
POST /api/v1/gpio
X-API-Key: <api_key>
Content-Type: application/json

{ "pin": 26, "value": 1 }
```

## Leer el estado

```http
GET /api/v1/gpio
```

```json
{ "gpio": [ { "id": "relay_riego", "pin": 26, "mode": "output", "value": 1 } ] }
```

## Expansión con MCP23017

- Un MCP23017 agrega **16 GPIO** por I²C (dirección `0x20`–`0x27`).
- Se inicializa automáticamente si algún `gpio[]` usa `expander_addr != 0`.
- Los pines del expander se numeran **0–15**.

## Consideraciones eléctricas

| Actuador | Nota |
|----------|------|
| Relé | Usar módulo de relé con optoacoplador y transistor; no cargar el pin directamente |
| LED | Resistencia limitadora (220–470 Ω) |
| Válvula/bomba | Relé + fuente externa |
| Sirena | Relé o transistor |

!!! warning "Nivel lógico"
    El ESP32 es **3,3 V**. Los actuadores de 5 V/12 V deben manejarse con
    transistor, MOSFET o módulo de relé; nunca conectes la carga directa al GPIO.

## Accionamiento automático (reglas)

Las salidas se pueden combinar con [reglas](Referencia-configuracion.md#rules) para
actuar por umbrales. El patrón es: un módulo externo lee las alarmas/mediciones y
escribe `POST /api/v1/gpio`, o una regla de alarma dispara una acción vía el
Servidor Central.

> El control **PID** de actuadores está previsto como mejora futura; hoy el control
> es por **umbrales** (reglas `gt`/`lt`/`ge`/`le`) e **histéresis**. Ver
> [Control PID](Control-PID.md).
