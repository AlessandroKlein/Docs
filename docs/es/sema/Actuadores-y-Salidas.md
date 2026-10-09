---
tags:
  - sema
  - actuadores
  - gpio
---

# Actuadores y salidas

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

SEMA controla **salidas digitales** (relés, LEDs, válvulas, sirenas, bombas) con el
subsistema de **GPIO standalone** (`GpioManager`), que maneja pines nativos del ESP32,
pines del expansor I²C **MCP23017** y —con `SEMA_USE_SHIFT`— **registros de
desplazamiento 74HC595**. Fuentes: `include/core/GpioManager.hpp`,
`src/core/GpioManager.cpp`, `include/core/ConfigManager.hpp` (`GpioSpec`),
`src/core/ShiftRegisterManager.cpp` y `src/core/web/HttpServer.cpp`.

## 1. Cómo se define una salida (`gpio[]`)

Cada salida es un objeto `GpioSpec` dentro del array `gpio` de la configuración:

```json
{
  "gpio": [
    { "id": "relay_riego", "pin": 26, "mode": "output", "initial": 0, "expander_addr": 0 },
    { "id": "led_estado",  "pin": 2,  "mode": "output", "initial": 1, "expander_addr": 0 },
    { "id": "valvula_2",   "pin": 3,  "mode": "output", "initial": 0, "expander_addr": 32 },
    { "id": "puerta",      "pin": 5,  "mode": "input_pullup", "initial": 0, "expander_addr": 32 }
  ]
}
```

| Clave | Tipo | Default | Significado | Origen |
|-------|------|---------|-------------|--------|
| `id` | string | `""` | Nombre lógico; se devuelve en `GET /api/v1/gpio` | REPO |
| `pin` | uint8 | `0` | GPIO nativo, o pin 0–15 del MCP23017 | REPO |
| `mode` | string | `"output"` al parsear (`"input"` en el `GpioSpec` por defecto) | `output` · `input` · `input_pullup` · `input_pulldown` | REPO |
| `initial` | uint8 | `0` | Estado que se escribe al aplicar la config (solo si `mode == "output"`) | REPO |
| `expander_addr` | uint8 | `0` | `0` = pin nativo; distinto de 0 = MCP23017 en esa dirección I²C | REPO |

Detalles de implementación (`GpioManager::apply`):

1. Reinicia `specs_` y la detección del expansor.
2. Busca **la primera** entrada con `expanderAddr != 0` e intenta `mcp_.begin_I2C(addr)`.
   Si hay varias direcciones distintas en `gpio[]`, se usa **una sola** (la primera):
   no se soportan dos MCP23017 simultáneos.
3. Si el expansor no responde (`mcpReady_ == false`), las entradas con
   `expander_addr != 0` **se ignoran silenciosamente**; solo se configuran las nativas.
4. Cualquier `mode` que no sea `output` / `input_pullup` / `input_pulldown` cae en
   `INPUT` (el `input` explícito y cualquier valor inválido).

> ⚠️ `GpioManager::find(pin)` resuelve por número de pin, no por `id` ni por expansor:
> **no repitas el mismo `pin` en dos entradas** (tampoco un pin nativo y un pin del
> MCP23017 a la vez).

## 2. Tipos de salida

| Tipo | Estado | Detalle | Origen |
|------|--------|---------|--------|
| Digital (ON/OFF) | ✅ Implementado | `pinMode(pin, OUTPUT)` + `digitalWrite(pin, value ? HIGH : LOW)` | REPO |
| Entrada digital | ✅ Implementado | `input`, `input_pullup`, `input_pulldown` | REPO |
| Salida por MCP23017 | ✅ Implementado | `mcp_.pinMode()` / `mcp_.digitalWrite()` (pines 0–15) | REPO |
| Salida por MCP23S17 (SPI) | ⚠️ Solo configuración | `mcp23s17_cs` + `mcp23s17_pins[16]` se guardan; no hay driver que los escriba | REPO |
| PWM por LEDC | ❌ **No implementado** | Existe la capacidad declarada `ledc_pwm` (`include/core/Capability.hpp`) y el README §29 lo plantea como objetivo, pero **no hay `ledcSetup`/`ledcWrite` en el firmware** | REPO |
| Soft-PWM por 74HC595 | ❌ No implementado | README §29 lo describe como arquitectura futura (buffer + DMA/timer) | REPO |
| Control PID | ❌ No implementado | Ver [Control PID](Control-PID.md) | REPO |

> **Verificado:** en todo el repositorio de firmware no aparece `ledc`, `analogWrite`
> ni ninguna API PWM. El único camino de salida es digital (`HIGH`/`LOW`).
> La estrategia de PWM disponible hoy es **externa**: un módulo que reciba la orden
> digital y genere el PWM, o un actuador con entrada de contacto seco.

## 3. API REST de salidas

### 3.1 Leer el estado

```http
GET /api/v1/gpio
```

```json
{
  "gpio": [
    { "id": "relay_riego", "pin": 26, "mode": "output", "value": 1 },
    { "id": "valvula_2",   "pin": 3,  "mode": "output", "value": 0 }
  ]
}
```

El valor se lee con `GpioManager::read(pin)`: si el pin pertenece al expansor y está
listo, lee del MCP23017; si no, hace `digitalRead()` del pin nativo.

### 3.2 Escribir una salida

```http
POST /api/v1/gpio
Content-Type: application/json

{ "pin": 26, "value": 1 }
```

Respuesta: `{"ok":true}`. Claves que acepta el handler: **solo** `pin` (uint8) y
`value` (int; `0` = LOW, distinto de 0 = HIGH). No acepta `id`, ni `expander`, ni
valores analógicos.

| Aspecto | Comportamiento | Origen |
|---------|----------------|--------|
| Autenticación de escritura | `webAuthed()`; si no hay contraseña web configurada, el endpoint queda **abierto** | REPO |
| Autenticación de lectura | `GET /api/v1/gpio` **no** chequea autenticación | REPO |
| Error si falta body | `400 {"error":"missing body"}` | REPO |
| Error si el JSON es inválido | `400 {"error":"invalid json"}` | REPO |
| Pin inexistente en `gpio[]` | Se escribe igual con `digitalWrite()` sobre el pin nativo (no valida que esté declarado) | REPO |

### 3.3 Configurar las salidas

```http
POST /api/v1/config/io
Content-Type: application/json

{ "gpio": [ { "id": "relay", "pin": 26, "mode": "output", "initial": 0, "expander": 0 } ] }
```

> ⚠️ **Discrepancia de claves:** en `POST /api/v1/config/io` el campo del expansor se
> llama **`expander`**, mientras que el JSON que devuelve `GET /api/v1/config` y el
> que se guarda en NVS lo llaman **`expander_addr`**. Si mandás `expander_addr` por
> este endpoint, se guarda como `0` (pin nativo).
> Además, la página web `/config/sensors` **siempre envía `gpio: []`** en
> `saveIo()`: guardar desde esa página **borra todas las salidas configuradas**.
> La única vía soportada hoy para dar de alta salidas es la API REST.

## 4. Expansores

| Expansor | Interfaz | Direcciones | Pines | Estado | Origen |
|----------|----------|-------------|-------|--------|--------|
| **MCP23017** | I²C | 0x20–0x27 (config `expander_addr` en decimal: 32 = 0x20) | 16 (0–15) | ✅ Implementado en `GpioManager` | REPO |
| **MCP23S17** | SPI | CS configurable (`mcp23s17_cs`) | 16 (A0–A7 / B0–B7) | ⚠️ Solo configuración (sin driver de escritura) | REPO |
| **74HC595** | SPI bit-banged (MOSI + SCK + LATCH propio) | — | 8 por chip, encadenables | ⚠️ `SEMA_USE_SHIFT` **deshabilitado** en los 3 entornos | REPO |
| **74HC165** | SPI bit-banged | — | 8 entradas por chip | ⚠️ Idem | REPO |
| **MAX14830 / SC18IS602B** | SPI | CS configurable | 4 UART / bus I²C | ⚠️ Solo configuración (el sensor puede apuntar con `uart`/`bus`, pero ningún driver lo usa) | REPO |

### 4.1 MCP23017 (I²C)

- Se inicializa automáticamente si **algún** `gpio[]` tiene `expander_addr != 0`.
- Los pines del expansor se numeran **0–15** dentro de `pin` (no 100+ como en la UI,
  que usa 100–115 solo como marcador visual en los selectores de pin).
- La dirección se escribe en decimal en la config: `32` = `0x20`, `33` = `0x21`, …
- La librería es `Adafruit_MCP23X17` (`Adafruit MCP23017 Arduino Library@^2.3.0`).

### 4.2 Registros de desplazamiento (74HC595 / 74HC165)

Con `SEMA_USE_SHIFT=1` (no está activado en `esp32doit-devkit-v1`,
`esp32-s3-devkitc-1` ni `esp32-wroom-32u`), `ShiftRegisterManager` escribe/lee **un byte
por vez en todos los chips del mismo tipo**:

| Aspecto | Comportamiento | Origen |
|---------|----------------|--------|
| Pines usados | DAT = `SEMA_SPI_MOSI`, CLK = `SEMA_SPI_SCK`, LATCH propio de cada chip (`latch_pin`) | REPO |
| Escritura | `writeByte(value)` recorre **todos** los 74HC595 y escribe el mismo byte (no hay direccionamiento por chip) | REPO |
| Lectura | `readByte()` lee **el primer** 74HC165 de la lista | REPO |
| Cascada | El README §27 describe la cascada (2 chips = 16 salidas, 4 = 32, 8 = 64), pero el driver actual escribe el mismo byte en cada chip: la cascada real **no está implementada** | REPO |
| API | `GET /api/v1/shift` → `{"value":N}` · `POST /api/v1/shift` `{"value":N}` (requiere auth) | REPO |
| Tipo de chip | `"74HC595"` (cualquier valor distinto de `"74HC165"` se trata como salida) | REPO |

> ⚠️ En los entornos compilados por defecto, `/api/v1/shift` devuelve
> `{"value":0}` y las escrituras no hacen nada, porque `SEMA_USE_SHIFT` está en `0`.
> Ver [Expansores de entrada/salida](Expansores-de-entrada-salida.md).

## 5. Consideraciones eléctricas

| Tema | Regla | Origen |
|------|-------|--------|
| Nivel lógico | El ESP32 y todos los buses son de **3,3 V**. Un GPIO no debe recibir 5 V | REPO |
| Corriente por pin | El pin de un GPIO del ESP32 maneja corrientes chicas (≤ 12 mA como criterio conservador). Para relés, bobinas, motores o tiras LED, usar transistor/MOSFET o módulo | REC |
| Relé | Usar **módulo de relé con transistor + optoacoplador**; nunca conectar la bobina directo al GPIO | REPO (página previa del repo Docs) |
| Diodo de flyback | `1N4148` (bobina chica) o `1N4007` (bobina mediana) en paralelo con la bobina, cátodo al positivo, si manejás el relé con un transistor propio | REC |
| Optoacoplamiento | Preferible en módulos de relé y en entradas de campo: separa el GND de la carga del GND lógico | REC |
| Aislamiento galvánico | El bus RS485 usa `TD501D485H` aislado (ver [Hardware y conexiones](Hardware-y-Conexiones.md) §7) | REPO |
| Alimentación de la carga | Fuente externa separada para válvulas/bombas/sirenas; **GND común** con el ESP32 (salvo que el módulo esté optoacoplado) | REC |
| Cargas inductivas | Snubber RC (100 nF + 100 Ω) en paralelo con contactos que conmutan cargas inductivas de AC | REC |
| Contacto seco | Para cargas de 220 V AC, el relé debe estar correctamente dimensionado y la parte de potencia **nunca** compartir pista con la lógica | REC |
| Arranque seguro | `initial` se aplica antes de que el resto del sistema arranque: usá `0` para que un relé de riego no quede energizado tras un reinicio | REPO |
| Entradas con pull | `input_pullup` / `input_pulldown` usan los pull internos del ESP32 (~45 kΩ). Para cables largos, agregá un pull externo de 4,7 kΩ–10 kΩ y un capacitor de 100 nF | REC |
| Pines solo-entrada | GPIO 34–39 no sirven como salida; el GPIO de batería por defecto (34) es de ese grupo | REPO |

> Los valores de diodos, snubbers y pull externos **no** están fijados por el firmware:
> son recomendaciones de instalación. Lo único que el repo fija es la tensión (3,3 V),
> el modo (`output`/`input*`) y el estado inicial (`initial`).

## 6. Accionamiento automático por reglas

Hoy **no existe** una acción de salida dentro del motor de reglas: `RuleEngine`
(`src/core/alarms/RuleEngine.cpp`) evalúa umbrales y publica alarmas, pero no escribe
GPIO. El accionamiento se arma en dos capas:

| Capa | Qué hace | Cómo |
|------|----------|------|
| Umbral y alarma | Reglas `gt` / `lt` / `ge` / `le` sobre `sensorId` + `channelId` con `value` | Config `rules[]` o `PUT /api/v1/config`; ver [Alarmas y reglas](Alarmas-y-reglas.md) |
| Acción física | Un cliente (script, Home Assistant, Servidor Central, Node-RED) lee `GET /api/v1/alarms`, `GET /api/v1/sensors` o `GET /api/v1/events` y ejecuta `POST /api/v1/gpio` | API REST |

```mermaid
flowchart LR
    S["Sensores (cada 10 s)"] --> R["RuleEngine (umbrales)"]
    R --> A["Alarmas / eventos"]
    A --> C["Cliente externo (poll o MQTT)"]
    C -->|"POST /api/v1/gpio"| G["GpioManager"]
    G --> P["Pin nativo / MCP23017"]
```

Regla por defecto si `rules[]` está vacío: `high_temp` sobre `EXT`/`temperature`
con umbral `> 40 °C` (no acciona ninguna salida por sí sola).

> ⚠️ **No implementado:** publicación MQTT de comandos de salida, histéresis
> automática, enclavamientos, temporizadores y control PID. Si los necesitás, la lógica
> vive en el cliente externo. Ver [Control PID](Control-PID.md) y
> [Mejoras y roadmap](Mejoras-y-roadmap.md).

## 7. Ejemplo completo de instalación

```text
Config gpio[]            → relay_riego (pin 26, output, initial 0)
                           + sensor de humedad de suelo (ADS1115 A0)
Regla rules[]            → soil_dry: SOIL_MOIST/soil_moisture < 25
Cliente externo          → GET /api/v1/alarms cada 30 s
                           si soil_dry activa → POST /api/v1/gpio {"pin":26,"value":1}
                           al cerrar la válvula → POST /api/v1/gpio {"pin":26,"value":0}
```

---

## Ver también

- [Alarmas y reglas](Alarmas-y-reglas.md) · [Expansores de entrada/salida](Expansores-de-entrada-salida.md)
- [Hardware y conexiones](Hardware-y-Conexiones.md) · [Guía de pines](Guia-de-pines.md) ·
  [Sensores](Sensores.md) · [Materiales](Materiales.md)
- [API REST](API-REST.md) · [Control PID](Control-PID.md) · [Configuración](Configuracion.md)
