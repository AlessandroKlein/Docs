---
tags:
  - sema
  - hardware
  - expansores
---

# Expansores de entrada/salida

> **Tipo:** Referencia | **Estado:** En desarrollo | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Qué expansores de E/S contempla SEMA, cuáles están **implementados de verdad** y cuáles
son solo configuración. Verificado contra `include/core/ConfigManager.hpp`,
`src/core/GpioManager.cpp`, `src/core/ShiftRegisterManager.cpp`,
`src/core/sensors/Ads1115Sensor.cpp`, `src/core/SemaCore.cpp` y
`src/core/web/HttpServer.cpp`.

> Los valores eléctricos marcados **(datasheet)** provienen de la hoja de datos del
> fabricante; el repositorio SEMA no los declara.

## 1. Panorama

| Expansor | Interfaz | Canales | Clave JSON | Estado real |
|----------|----------|---------|------------|-------------|
| **MCP23017** | I²C | 16 (A0-A7, B0-B7) | `gpio[].expander_addr` | ✅ Implementado (`GpioManager`), sin editor web |
| **MCP23S17** | SPI | 16 (A0-A7, B0-B7) | `mcp23s17_cs`, `mcp23s17_pins[16]` | ❌ Solo configuración: no hay driver |
| **74HC595** | SPI (bit-bang: MOSI+SCK+LATCH) | 8 salidas por chip | `shift_registers[].{type,latch,pins}` | ⚠️ Driver escrito pero **compilado fuera** (`SEMA_USE_SHIFT=0` en los 4 entornos) |
| **74HC165** | SPI (bit-bang: MOSI+SCK+LATCH) | 8 entradas por chip | `shift_registers[].{type,latch,pins}` | ⚠️ Igual que el 74HC595 |
| **ADS1115** | I²C | 4 (A0-A3, 16 bits) | `sensors[model=ADS1115].{pin,channel,scale,offset}` | ✅ Implementado, pero como **sensor**, no como GPIO |
| **MAX14830** | SPI → 4 UART | 4 puertos serie | `spi_expanders[].{type,cs}` | ❌ Solo configuración: no hay driver |
| **SC18IS602B** | SPI → I²C | 1 bus I²C puenteado | `spi_expanders[].{type,cs}` | ❌ Solo configuración: no hay driver |

## 2. MCP23017 (I²C) — implementado

| Aspecto | Detalle |
|---------|---------|
| Clases | `include/core/GpioManager.hpp`, `src/core/GpioManager.cpp` |
| Librería | `adafruit/Adafruit MCP23017 Arduino Library@^2.3.0` (tipo interno `Adafruit_MCP23X17`) |
| Inicialización | `GpioManager::apply()` recorre los `GpioSpec` y toma el **primer** `expanderAddr != 0`; con ese valor llama `mcp_.begin_I2C(addr)` y marca `mcpReady_` (`GpioManager.cpp:22-29`) |
| Canales | 16: `pin` 0-7 = A0-A7, `pin` 8-15 = B0-B7 |
| Modos | `output`, `input`, `input_pullup`, `input_pulldown` (cualquier otro valor cae a `INPUT`) |
| Estado inicial | En `output` se escribe `gpio[].initial` (0/1) al aplicar la config |
| Lectura/escritura | `gpio.read(pin)` y `gpio.write(pin, value)` enrutan al chip si el pin tiene `expanderAddr != 0` y el chip respondió |
| Direcciones | 0x20-0x27 según A0-A2 (datasheet) |
| Config desde la web | ❌ **No hay editor**: la página de E/S envía `gpio: []` (`HttpServer.cpp:1271`). Solo se configuran por `PUT /api/v1/config` (clave `gpio[].expander_addr`) o por `POST /api/v1/config/io` (clave `gpio[].expander`, nombre distinto) |
| Límite verificado | Con dos MCP23017 de direcciones distintas, `apply()` solo inicializa el primero pero luego escribe **todos** los pines contra ese mismo objeto: el segundo chip nunca recibe nada |
| `input_pulldown` | ⚠️ El MCP23017 solo tiene pull-ups internos (datasheet): el modo se acepta en la config pero no garantiza un pull-down real |

### 2.1 Ejemplo de configuración (PUT `/api/v1/config`)

```json
{
  "gpio": [
    { "id": "riego",  "pin": 0, "mode": "output", "initial": 0, "expander_addr": 32 },
    { "id": "puerta", "pin": 8, "mode": "input_pullup", "initial": 0, "expander_addr": 32 }
  ]
}
```

`expander_addr: 32` es `0x20` en decimal: el JSON guarda la **dirección I²C**, no un
índice de chip. Después de aplicar hay que reiniciar o volver a llamar
`core_->applyGpio()` (lo hace `POST /api/v1/config/io`).

## 3. MCP23S17 (SPI) — solo configuración

| Aspecto | Detalle |
|---------|---------|
| Claves JSON | `mcp23s17_cs` (0 = no usar), `mcp23s17_pins[16]` con `0` = no usado, `1` = salida, `2` = entrada |
| Endpoint | `POST /api/v1/config/io` |
| Web | Página de E/S: selector de CS + 16 selectores A0-A7/B0-B7 (`HttpServer.cpp:1204-1213`) |
| Estado real | ❌ **No hay driver**: `mcp23s17_cs` y `mcp23s17_pins` solo se guardan, se serializan y se muestran. Nada los escribe al hardware |
| Efecto en la web | Los 16 pines aparecen en el selector de pines como pseudo-GPIO `100..115` («MCP A0» … «MCP B7») y se pueden asignar a otros chips, aunque el chip no exista en el firmware |
| Modo demo | Con `SEMA_DEMO=1`, si `mcp23s17_cs` es 0 se fuerza a **5** y los 16 pines a salida para que la UI muestre el expansor (`ConfigManager.cpp:385-391`) |

## 4. 74HC595 / 74HC165 (registros de desplazamiento) — driver presente, compilado fuera

| Aspecto | Detalle |
|---------|---------|
| Clases | `include/core/ShiftRegisterManager.hpp`, `src/core/ShiftRegisterManager.cpp` |
| Flag | `SEMA_USE_SHIFT`, por defecto **`SEMA_OFF`** (`HwProfile.hpp:68-70`); **ningún entorno de `platformio.ini` lo activa** |
| Efecto del flag | `SemaCore` solo llama `shift_.apply(...)` bajo `#if SEMA_USE_SHIFT` (`SemaCore.cpp:124-126` y `321-325`), y `POST /api/v1/config/io` solo parsea `shift_registers` bajo el mismo `#if` (`HttpServer.cpp:1595-1610`) |
| Consecuencia | Con los cuatro entornos actuales, `cfg_` queda vacío: `writeByte()` no hace nada y `readByte()` devuelve `0`. `GET /api/v1/system → shift_enabled` informa `false` y la web **oculta** el editor |
| Pines usados | `SEMA_SPI_MOSI` (DAT) y `SEMA_SPI_SCK` (CLK) como bit-bang, más un **LATCH propio por chip** (`shift_registers[].latch_pin`) |
| Sentido | `pinMode(SEMA_SPI_MOSI, tipo == "74HC165" ? INPUT : OUTPUT)` y `pinMode(SEMA_SPI_SCK, OUTPUT)` |
| Escritura | `writeByte(value)`: por **cada** 74HC595 configurado hace `LATCH=LOW` → `shiftOut(MOSI, SCK, MSBFIRST, value)` → `LATCH=HIGH`. El mismo byte va a todos los chips |
| Lectura | `readByte()`: recorre la lista y devuelve el byte del **primer** 74HC165 (`SH/LD=LOW`, espera 5 µs, `SH/LD=HIGH`, `shiftIn`). No lee los siguientes |
| Alcance real | ❌ **No hay cascada**: a pesar del comentario «Se pueden encadenar varios» (`ConfigManager.hpp:131`), no se envían varios bytes ni se leen varios chips |
| `pinModes[8]` | Se guarda por chip (Q0-Q7 / D0-D7) pero **ningún código lo lee**: es informativo |

### 4.1 Para activarlo

```ini
; platformio.ini → build_flags del entorno
-D SEMA_USE_SHIFT=1
```

Con eso se compilan `apply()`, el parseo de `shift_registers` y el editor web. Antes de
usarlo en producción hay que resolver la cascada y los niveles lógicos (el propio
`HwProfile.hpp:66-67` lo marca como **beta**).

## 5. ADS1115 (ADC I²C de 16 bits) — implementado como sensor

| Aspecto | Detalle |
|---------|---------|
| Clase | `src/core/sensors/Ads1115Sensor.cpp` (librería `adafruit/Adafruit ADS1X15@^2.0.0`) |
| Interfaz declarada | `"I2C"` (por eso aparece en el bloque I²C de la página de sensores, no en el de analógicos nativos) |
| Canales | 4 single-ended: `sensors[].pin` = canal 0-3 (A0-A3) |
| Dirección | `ads1115.begin()` **sin** dirección → 0x48 fijo. El selector «ID» de la web se ignora |
| Nombre del canal | `sensors[].channel` (p. ej. `wind_direction`, `voltage`) |
| Escalado | `valor = raw × scale + offset`, con `raw` de 16 bits con signo |
| Instancia | `static Adafruit_ADS1115 ads1115` compartida por todas las entradas del catálogo: la última `begin()` define el chip |
| Uso desde la web | En cada fila analógica hay un selector `FUENTE`: `ADC ESP32` (usa `pin`) o `ADS1115` (usa `pin` como canal). Elegir ADS1115 cambia el `model` del JSON a `ADS1115` (`HttpServer.cpp:1079`, `1120-1134`) |
| Límite | No hay soporte de modo diferencial, ni de *data rate*, ni de PGA: se usan los defaults de la librería (FSR de ±6,144 V, datasheet) |

## 6. MAX14830 (4 UART por SPI) — solo configuración

| Aspecto | Detalle |
|---------|---------|
| Clave JSON | `spi_expanders[]` con `{ "type": "MAX14830", "cs": <GPIO> }` |
| Dónde se configura | Página de E/S → `POST /api/v1/config/io` |
| Uso en los sensores | Si existe un MAX14830 con `cs != 0`, los selectores `UART` de las filas UART y de Modbus/Zigbee ofrecen `MAX14830 CS=<cs> U0..U3` (`HttpServer.cpp:1044-1053`) |
| Estado real | ❌ **No hay driver**: `src/core/` no contiene ninguna referencia al chip fuera de `ConfigManager` y `HttpServer`. No se inicializa el SPI ni se crean los 4 puertos |
| Además | `POST /api/v1/config/sensors` **no** copia `uart` ni `uart_port` al `SensorSpec` (`HttpServer.cpp:1547-1563`): la selección del puerto se descarta al guardar. Solo sobrevive si se escribe con `PUT /api/v1/config` |
| Falta en la config | No hay campo para el pin de interrupción (`/IRQ`) del MAX14830: un driver real necesitaría polling o un GPIO adicional |

## 7. SC18IS602B (I²C por SPI) — solo configuración

| Aspecto | Detalle |
|---------|---------|
| Clave JSON | `spi_expanders[]` con `{ "type": "SC18IS602B", "cs": <GPIO> }` |
| Uso en los sensores | El selector `Bus` de las filas I²C ofrece `Nativo (SDA/SCL)` o `SC18IS602B (CS=<cs>)` (`HttpServer.cpp:1039-1043`), que escribe `sensors[].bus` |
| Estado real | ❌ **No hay driver**: `sensors[].bus` se guarda y se serializa, pero ningún driver I²C lo lee (los 12 drivers llaman `Wire.begin()` directo) |
| Mismo problema | `POST /api/v1/config/sensors` tampoco copia `bus` |

## 8. Endpoints

| Endpoint | Método | Autenticación | Cuerpo / respuesta |
|----------|--------|---------------|--------------------|
| `/api/v1/gpio` | GET | No | `{"gpio":[{"id","pin","mode","value"}]}` — un objeto por `GpioSpec`; `value` sale de `gpio.read(pin)` (`HttpServer.cpp:2404-2418`) |
| `/api/v1/gpio` | POST | Sí (`webAuthed`) | `{"pin": <uint8>, "value": <int>}` → `{"ok":true}`. Escribe sin validar que el pin exista en la config |
| `/api/v1/shift` | GET | No | `{"value": <0-255>}` — byte del primer 74HC165 (0 si el flag está apagado) |
| `/api/v1/shift` | POST | Sí | `{"value": <0-255>}` → `{"ok":true}`; el byte va a todos los 74HC595 |
| `/api/v1/config/io` | POST | Sí | `mcp23s17_cs`, `mcp23s17_pins[16]`, `shift_registers[{type,latch,pins[8]}]`, `spi_expanders[{type,cs}]`, `gpio[{id,pin,mode,initial,expander}]` → `{"ok":true}` |
| `/api/v1/config` | GET | Sí | Serializa todo: `gpio[].expander_addr`, `mcp23s17_cs`, `mcp23s17_pins`, `shift_registers[].latch_pin`, `spi_expanders[].{type,cs}` |
| `/api/v1/config` | PUT | Sí | Igual que el GET, pero `gpio[].expander_addr` (no `expander`) |
| `/api/v1/system` | GET | No | `shift_enabled` (si `SEMA_USE_SHIFT != 0`), `reserved_pins[]`, `spi_sck`, `spi_miso`, `spi_mosi`, `sd_cs` |

### 8.1 Diferencia de clave entre los dos caminos del JSON

| Campo | `PUT /api/v1/config` (`ConfigManager::parseInto`) | `POST /api/v1/config/io` (`onConfigIo`) |
|-------|---------------------------------------------------|------------------------------------------|
| Dirección I²C del MCP23017 | `gpio[].expander_addr` | `gpio[].expander` |

Un cliente que reenvíe el JSON del `GET /api/v1/config` al `POST /api/v1/config/io`
perderá la dirección del expansor y los pines caerán a GPIO nativo.

## 9. Fichas de conexión

### 9.1 MCP23017 (16 E/S por I²C)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | I²C, dirección 0x20-0x27 según A0/A1/A2 |
| Tensión | 1,8-5,5 V (el tooltip de la web lo declara); lógica del bus a 3,3 V |
| Resistencia | Pull-up de 4,7 kΩ en SDA/SCL y pull-up de 10 kΩ en `/RESET` a VDD (datasheet) |
| Capacitor | 100 nF cerámico entre VDD y GND (datasheet) |
| Conexión | VDD→3,3 V · GND→GND · SDA→GPIO21 · SCL→GPIO22 · A0-A2→según dirección deseada (`expander_addr`) · GPA0-GPA7 y GPB0-GPB7→carga |
| Notas | Salidas de 25 mA por pin (datasheet): para relés conviene un driver (ULN2803 o transistor) y diodo de rueda libre |

### 9.2 MCP23S17 (16 E/S por SPI)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | SPI (hasta 10 MHz, datasheet), CS propio (`mcp23s17_cs`) |
| Tensión | 1,8-5,5 V (datasheet); lógica 3,3 V |
| Resistencia | Pull-up de 10 kΩ en `/RESET` y en CS (datasheet) |
| Capacitor | 100 nF entre VDD y GND (datasheet) |
| Conexión | SI→SEMA_SPI_MOSI · SO→SEMA_SPI_MISO · SCK→SEMA_SPI_SCK · CS→GPIO elegido · A0-A2→dirección del chip en el bus |
| Notas | ❌ Sin driver en el firmware: la conexión es la correcta pero no se inicializa |

### 9.3 74HC595 (8 salidas serie)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | Serie: DS (datos), SHCP (reloj), STCP (latch/RCLK), OE, MR |
| Tensión | 2,0-6,0 V; a 3,3 V los niveles de entrada son válidos |
| Resistencia | 10 kΩ de MR (`SRCLR`) a VCC; 100 Ω en serie por salida si maneja líneas largas o LEDs (datasheet) |
| Capacitor | 100 nF entre VCC y GND junto al integrado (datasheet) |
| Conexión | DS→SEMA_SPI_MOSI · SHCP→SEMA_SPI_SCK · STCP→`shift_registers[].latch_pin` · OE→GND (habilita salidas) · MR→VCC (sin reset) · Q0-Q7→carga · VCC→3,3 V |
| Notas | Si el módulo se alimenta a 5 V, sus entradas toleran 3,3 V pero **no** al revés: no conectar salidas de 5 V al ESP32 |

### 9.4 74HC165 (8 entradas serie)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | Serie: D0-D7 (paralelo), CP (reloj), PL (`SH/LD`), CE, Q7 |
| Tensión | 2,0-6,0 V; con VCC = 3,3 V las entradas son seguras para el ESP32 |
| Resistencia | Pull-up o pull-down de 10 kΩ en cada D0-D7 que quede flotante (datasheet) |
| Capacitor | 100 nF entre VCC y GND (datasheet) |
| Conexión | Q7→SEMA_SPI_MISO · CP→SEMA_SPI_SCK · PL→`shift_registers[].latch_pin` · CE→GND · D0-D7→entradas |
| Notas | ⚠️ Compilado fuera por defecto (`SEMA_USE_SHIFT=0`); además `readByte()` solo lee el primer chip |

### 9.5 ADS1115 (ADC 16 bits por I²C)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | I²C, 0x48 por defecto (ADDR a GND) |
| Tensión | 2,0-5,5 V (datasheet); lógica 3,3 V |
| Resistencia | Pull-up de 4,7 kΩ en SDA/SCL; divisor o acondicionador en la entrada si la señal supera VDD |
| Capacitor | 100 nF entre VDD y GND (datasheet); 100 nF opcional de filtro en cada entrada |
| Conexión | VDD→3,3 V · GND→GND · SDA→GPIO21 · SCL→GPIO22 · A0-A3→señal analógica |
| Notas | Entrada nunca por encima de VDD+0,3 V (datasheet). El firmware siempre usa 0x48 |

### 9.6 MAX14830 (4 UART por SPI)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | SPI (hasta 26 MHz, datasheet) + `/IRQ` + 4 pares TX/RX + 4 pares RTS/CTS |
| Tensión | 3,0-3,6 V (datasheet) |
| Resistencia | Pull-up de 10 kΩ en CS y en `/RESET` (datasheet) |
| Capacitor | 100 nF por alimentación + 10 µF de bulk; cristal de 3,6864 MHz con sus 2 × 22 pF (datasheet) |
| Conexión | SI/SO/SCK→bus SPI (`SEMA_SPI_MOSI`/`MISO`/`SCK`) · CS→`spi_expanders[].cs` · TX/RX de cada puerto→transceiver o sensor serie |
| Notas | ❌ Sin driver en el firmware. La config no tiene campo de `/IRQ`, imprescindible para recepción por interrupción |

### 9.7 SC18IS602B (puente SPI → I²C)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | SPI (esclavo) + bus I²C maestro hacia los dispositivos |
| Tensión | 2,4-3,6 V (datasheet) |
| Resistencia | Pull-up de 4,7 kΩ en SDA/SCL del bus puenteado y de 10 kΩ en CS (datasheet) |
| Capacitor | 100 nF entre VDD y GND (datasheet) |
| Conexión | SI/SO/SCK→bus SPI de SEMA · CS→`spi_expanders[].cs` · SDA/SCL del puente→sensores I²C |
| Notas | ❌ Sin driver en el firmware: `sensors[].bus` (el CS del puente) se guarda pero ningún driver lo usa |

---

## Ver también

- [Guía de pines](Guia-de-pines.md) · [Buses y periféricos](Buses-y-perifericos.md) · [Compatibilidad](Compatibilidad.md)
- [Actuadores y salidas](Actuadores-y-Salidas.md) · [Hardware y conexiones](Hardware-y-Conexiones.md) · [Sensores](Sensores.md)
- [Referencia de configuración](Referencia-configuracion.md) · [API REST](API-REST.md) · [Materiales](Materiales.md)
