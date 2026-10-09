---
tags:
  - sema
  - modulos
---

# Módulos y ciclo de vida

> **Tipo:** Referencia | **Estado:** En desarrollo | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Esta página responde una sola pregunta: **¿qué es un "módulo" en el código de SEMA
y cómo se activa?** La respuesta corta es que conviven dos definiciones distintas y
solo una está en uso.

## 1. El contrato `sema::Module`

`include/core/Module.hpp` define la interfaz mínima:

```cpp
enum class ModuleState : uint8_t {
  Available, Installed, Configured, Enabled, Running,
  Disabled, Uninstalled,
  InstallError, ConfigError, RuntimeError, UpdateError
};

class Module {
public:
  virtual ~Module() = default;
  virtual const char* id() const = 0;        // identificador estable
  virtual const char* version() const = 0;   // versión del módulo
  virtual bool install()   { return true; }
  virtual bool configure() { return true; }
  virtual bool enable()    { state_ = ModuleState::Enabled; return true; }
  virtual void start()     { state_ = ModuleState::Running; }
  virtual void loop()      {}
  virtual void stop()      { state_ = ModuleState::Enabled; }
  virtual bool disable()   { state_ = ModuleState::Disabled; return true; }
  ModuleState state() const { return state_; }
protected:
  ModuleState state_ = ModuleState::Available;
};
```

Observaciones verificadas sobre el contrato:

- **`id()` y `version()` son obligatorios** (puros); el resto tiene implementación
  por defecto.
- El enum va en PascalCase a propósito: evita colisionar con las macros `DISABLED`
  y `ENABLED` del core de Arduino/ESP32 (comentario en `Module.hpp`, líneas 12-13).
- **No existe `uninstall()`** ni `update()`: `Uninstalled` y `UpdateError` están en
  el enum pero ningún método los aplica.
- Los estados de error (`InstallError`, `ConfigError`, `RuntimeError`,
  `UpdateError`) no los asigna ningún código del repo.
- `enable()` no encadena nada: quien quiera pasar por `Installed`/`Configured` debe
  llamar a `install()` y `configure()` primero (y ninguno de los dos cambia el
  estado, son ganchos).

## 2. El registro `ModuleRegistry`

`include/core/ModuleRegistry.hpp` y `src/core/ModuleRegistry.cpp` (44 líneas)
implementan el registro:

| Método | Comportamiento real |
|--------|---------------------|
| `registerModule(Module&)` | Devuelve `false` si ya hay un módulo con el mismo `id()`; si no, agrega el puntero al vector |
| `find(const char* id)` | Búsqueda lineal por `strcmp` |
| `enableAll()` | Por cada módulo en estado `Available`: `install()` → `configure()` → `enable()` → `start()` |
| `loopAll()` | Por cada módulo en estado `Enabled` **o** `Running`: `loop()` |
| `count()` | Tamaño del vector |

`enableAll()` sí recorre la cadena completa del ciclo de vida; la implementación es
correcta y coincide con el diagrama del README §53.

## 3. La verdad incómoda: el registro está vacío

⚠️ **`ModuleRegistry::registerModule()` no se llama nunca en todo el árbol de
código de v1.103.0.**

Evidencia:

- `SemaCore.hpp` declara el miembro `ModuleRegistry modules_;` (línea 123) y lo
  expone con `modules()` (línea 84), pero en `SemaCore.cpp` solo se invocan
  `modules_.enableAll()` (línea 200), `modules_.loopAll()` (línea 368) y
  `modules_.count()` (línea 224, para el log de arranque).
- Ninguna clase hereda de `sema::Module`. La búsqueda de `public Module` / `: Module`
  en `.hpp` y `.cpp` no devuelve ninguna clase de SEMA derivada de esa interfaz.
- Consecuencia en runtime: `modules_.count()` es **0**, `enableAll()` no hace nada y
  `loopAll()` itera sobre un vector vacío. El serial de arranque imprime
  `Módulos registrados: 0`.

Cuidado con un falso positivo: `LoraManager` usa `new Module(cs, dio1, rst, busy)`
en `src/core/LoraManager.cpp` línea 26. Ese `Module` **no es `sema::Module`**: es la
clase de RadioLib que describe el módulo de radio SX1262. Son dos `Module`
homónimos de namespaces distintos.

### ¿Entonces el ciclo AVAILABLE → … → RUNNING está implementado o es conceptual?

| Aspecto | Estado |
|---------|--------|
| El enum `ModuleState` con los 11 valores | ✅ Implementado (`Module.hpp`) |
| El contrato `Module` con los ganchos de ciclo de vida | ✅ Implementado (`Module.hpp`) |
| El registro con `registerModule`/`find`/`enableAll`/`loopAll` | ✅ Implementado (`ModuleRegistry.cpp`) |
| El encadenado `Available → Installed → Configured → Enabled → Running` | ✅ Implementado **en `enableAll()`** |
| Aplicación real del ciclo a algún componente del firmware | ❌ **No implementado**: nadie registra módulos |

En resumen: el **mecanismo** está implementado y es funcional, pero el **uso** es
conceptual. Hoy ningún servicio se activa por esa vía.

## 4. Entonces, ¿qué existe realmente?

### 4.1 Servicios del núcleo (composición directa, no son módulos)

Son miembros de `SemaCore` (`include/core/SemaCore.hpp`, líneas 122-158). No
implementan `sema::Module`; se inicializan en el constructor o en `setup()`:

| Servicio | Rol | Ciclo de vida real |
|----------|-----|--------------------|
| `NvsStore store_` | Backend NVS (namespace `"sema"`) | Constructor → `begin("sema")` |
| `ModuleRegistry modules_` | Registro de módulos | Vacío siempre |
| `EventBus events_` | Publicación de eventos tipados | Constructor |
| `ConfigManager config_` | Configuración transaccional (schema 1) | Constructor → `load()` |
| `SensorManager sensors_` | Registro y lectura de drivers | `applySensors()` → `beginAll()` |
| `HistoryStore history_` | Histórico JSONL en SD | `begin()` + `enableSd()` opcional |
| `PublisherManager publishers_` | Enrutado a publicadores | `registerPublisher()` ×2 |
| `HttpPublisher webhook_` | POST HTTP a webhook | `setUrl()` con la config |
| `MqttPublisher mqtt_` | Publicación MQTT | `configure()` con la config |
| `RuleEngine rules_` | Evaluación de alarmas | `clear()` + `addRule()` |
| `DerivedCalculator derived_` | Derivadas extra y unidades | `configure(system)` |
| `EventLog eventLog_` | Log persistente de eventos | `begin()` (LittleFS) |
| `GpioManager gpio_` | Pines nativos y MCP23017 | `apply(cfg.gpio)` |
| `ShiftRegisterManager shift_` | 74HC595 / 74HC165 | `apply()` solo si `SEMA_USE_SHIFT` |
| `Watchdog watchdog_` | TWDT del ESP32 | `begin(10)` |
| `HealthMonitor health_` | Salud agregada y watchdog lógico | `begin()` |
| `Scheduler scheduler_` | Tareas periódicas cooperativas | `add()` ×2 |
| `WiFiManager wifi_` | STA/AP, mDNS, reconexión | `begin(...)` + `loop()` |
| `HttpServer http_` | Web + REST + WebSocket | `begin(*this)` + `loop()` |
| `PowerManager` (singleton) | Perfiles de energía, wake | `wakeReason()`, `enableRainWakeup()` |

### 4.2 Managers de bus/red compilados condicionalmente

Estos son los que la documentación suele llamar "módulos". Son miembros de
`SemaCore` incluidos con `#if SEMA_USE_*` y **se activan/desactivan por
configuración en runtime**, no por el `ModuleRegistry`:

| Manager | Flag de compilación | Clave de configuración | Driver / API usada | ¿Tarea propia? |
|---------|--------------------|------------------------|--------------------|----------------|
| `EthernetManager` | `SEMA_USE_ETHERNET` | `ethernet` (`enabled`, pines MDC/MDIO/PHY o W5500) | `ETH.begin()` (LAN8720A RMII) o `esp_eth` + `spi_master` (W5500) | No |
| `LoraManager` | `SEMA_USE_LORA` | `lora` (`enabled`, CS/RST/DIO1/BUSY, frecuencia, BW, SF, CR, potencia) | RadioLib `SX1262` | No (usa `startReceive()` por interrupción del driver) |
| `ModbusManager` | `SEMA_USE_MODBUS` | `modbus` (`enabled`, RX/TX/DE-RE, baud, slaveId, registro y cantidad) | `Serial2` + `ModbusMaster` | No |
| `CanManager` | `SEMA_USE_CAN` | `can` (`enabled`, TX/RX, velocidad) | driver TWAI (`twai_driver_install`, `twai_start`) | No (el driver IDF crea sus propias tareas internas) |
| `ZigbeeManager` | `SEMA_USE_ZIGBEE` | `zigbee` (`enabled`, RX/TX, baud) | `Serial1` + protocolo ZNP (tramas `0xFE` SOF) | No |
| `WiFiManager` | siempre (core) | `network` (`mode`, ssid, password, hostname, mdns, IP fija) | `WiFi`, `ESPmDNS` | No (la pila WiFi del IDF crea las suyas) |
| `GpioManager` | siempre (core) | `gpio[]` | `pinMode`/`digitalWrite` + Adafruit MCP23017 | No |
| `ShiftRegisterManager` | `SEMA_USE_SHIFT` (**0 por defecto**) | `shiftRegisters[]` | bit-banging sobre los pines SPI | No |

`SEMA_USE_SHIFT` está en **0** por defecto (`include/hw/HwProfile.hpp`, líneas
66-70): el driver de registros de desplazamiento es beta. Los endpoints
`/api/v1/shift` existen, pero con la compilación por defecto
`SemaCore::setup()` **no llama** a `shift_.apply(...)` (está bajo `#if SEMA_USE_SHIFT`).

### 4.3 Catálogo de sensores (17 modelos)

Los sensores **no** son módulos: son drivers que implementan `sema::Sensor`
(`include/core/sensors/Sensor.hpp`), con solo cuatro métodos (`id()`, `model()`,
`interface()`, `begin()`, `measure()`, `healthy()`), y los instancia
`SensorFactory::create(spec)`.

Modelos reconocidos por `SensorFactory` (los 17, en el orden del `if`):

| # | `spec.model` | Clase | Interfaz |
|---|--------------|-------|----------|
| 1 | `BME280` | `Bme280Sensor` | I²C |
| 2 | `SHT40` | `Sht40Sensor` | I²C |
| 3 | `SHT31` | `Sht31Sensor` | I²C |
| 4 | `BMP280` | `Bmp280Sensor` | I²C |
| 5 | `DS18B20` | `Ds18b20Sensor` | 1-Wire |
| 6 | `BH1750` | `Bh1750Sensor` | I²C |
| 7 | `AHT20` | `Aht20Sensor` | I²C |
| 8 | `ADC` | `AdcSensor` | ADC interno |
| 9 | `PCNT` | `PcntSensor` | PCNT (pulsos) |
| 10 | `VEML6075` | `Veml6075Sensor` | I²C |
| 11 | `SCD30` | `Scd30Sensor` | I²C |
| 12 | `SGP30` | `Sgp30Sensor` | I²C |
| 13 | `PMS5003` | `Pms5003Sensor` | UART |
| 14 | `AS3935` | `As3935Sensor` | I²C |
| 15 | `ADS1115` | `Ads1115Sensor` | I²C |
| 16 | `CO` | `CoSensor` | ADC |
| 17 | `SOLAR` | `SolarSensor` | ADC |

Cualquier otro string devuelve `nullptr` y `applySensors()` imprime
`Sensor desconocido: <id> (modelo <modelo>)`.

### 4.4 Solo configuración: sin driver propio

Los siguientes elementos existen **únicamente como claves de configuración** y como
opciones en la web. No hay una clase driver que los implemente:

| Elemento | Dónde aparece | Qué es |
|----------|---------------|--------|
| `MAX14830` | `SpiExpanderConfig::type` (`ConfigManager.hpp`) y `HttpServer.cpp` | Expansor UART por SPI: se declara tipo y chip-select |
| `SC18IS602B` | igual que el anterior, más `SensorSpec::bus` y `ModbusConfig::uart` / `ZigbeeConfig::uart` | Expansor I²C por SPI: se declara por su CS |
| `Mcp23s17Config` | `ConfigManager.hpp` (`csPin`, `pinModes[16]`) | Expansor GPIO por SPI. ⚠️ El único expansor GPIO **con driver** es el MCP23017 por I²C (`GpioManager`); el MCP23S17 solo está modelado en la configuración |
| `SEMA_PINS_FROM_FILE` | `platformio.ini` y `HwProfile.hpp`; aplicado por `applyHwProfile()` | Fija los pines de los **buses** (CAN, Modbus, Zigbee, LoRa, Ethernet) desde el archivo, pisando lo que venga de la web. **No** fija los pines del catálogo de sensores: eso lo hace `SEMA_FIXED_HARDWARE` (`BoardProfile.hpp`) |
| `SEMA_FIXED_HARDWARE` | `include/core/BoardProfile.hpp` | Fija los pines del **catálogo de sensores** (SDA, SCL, 1-Wire, ADC de batería) e ignora `config.sensors[]`. Por defecto vale 0 |

## 5. Cómo se activa y desactiva de verdad

El ciclo de vida real de un componente es:

```text
1. COMPILACIÓN   build_flags de platformio.ini → SEMA_USE_*
                 (si el flag es 0, el código no entra en el binario)
2. CONSTRUCCIÓN  el manager es un miembro de SemaCore; existe siempre
3. APLICACIÓN    apply(cfg) en setup() o en caliente (applyX()); si cfg.enabled
                 es false, el manager queda en ready_ = false y no hace nada
4. MARCHA        el manager solo participa si SemaCore::loop() lo llama
                 explícitamente (zigbee_.loop(), ethernet_.loop(), wifi_.loop())
5. RECONFIG      applyX() vuelve a aplicar la configuración sin reiniciar
```

### 5.1 Aplicación en caliente (sin reinicio)

`SemaCore` expone estos métodos públicos (`include/core/SemaCore.hpp`, líneas
61-82); son los que invoca `HttpServer` cuando se guarda configuración desde la web:

| Método | Qué reaplica |
|--------|--------------|
| `applyRules()` | Reglas de alarma (con `clear()` previo) |
| `applyCalibrations()` | Calibraciones por canal (con `clearCalibrations()` previo) |
| `applyGpio()` | Pines GPIO y expansor MCP23017 |
| `applyPublishers()` | URL del webhook y parámetros MQTT |
| `applySensors()` | Catálogo completo: borra, destruye los sensores propios y recrea |
| `applyShift()` | Registros de desplazamiento (bajo `SEMA_USE_SHIFT`) |
| `applyModbus()` | RS485/Modbus (bajo `SEMA_USE_MODBUS`) |
| `applyCan()` | TWAI (bajo `SEMA_USE_CAN`) |
| `applyLora()` | SX1262 (bajo `SEMA_USE_LORA`) |
| `applyZigbee()` | CC2652P2 (bajo `SEMA_USE_ZIGBEE`) |
| `applyEthernet()` | LAN8720A o W5500 (bajo `SEMA_USE_ETHERNET`) |

`applySensors()` merece atención: destruye los `Sensor*` que creó la factoría
(`ownedSensors_`) y vuelve a llamar a `SensorFactory::create()` por cada entrada
habilitada. Los sensores del **catálogo fijo** (`static Bme280Sensor bme280(...)`,
etc.) no se destruyen porque son objetos estáticos. Cambiar la configuración de
sensores desde la web, por lo tanto, **no requiere reinicio**; cambiar pines de
buses sí puede requerirlo según el driver.

⚠️ `applySensors()` se llama en `setup()` (línea 119) antes de que
`HttpServer::begin()` registre las rutas, y `derived_.configure()` se llama dos
veces (líneas 120 y 271, la segunda dentro de `applySensors()`).

## 6. Máquina de estados real de un componente

```mermaid
stateDiagram-v2
    [*] --> Compilado: SEMA_USE_X = 1
    Compilado --> Inerte: cfg.enabled = false
    Compilado --> Listo: apply(cfg) con enabled = true
    Inerte --> Listo: applyX() desde la web
    Listo --> EnServicio: SemaCore::loop() lo llama
    Listo --> Inerte: applyX() con enabled = false
    EnServicio --> Listo: deja de llamarse / falla el driver
    note right of Inerte
        ready_ = false
        (patrón común de
        Modbus/Can/Lora/Zigbee/Ethernet)
    end note
    note right of EnServicio
        Ethernet::loop() y Modbus::read()
        son no-op salvo en Zigbee
    end note
```

Comparación con el ciclo del `ModuleRegistry` (`Available → Installed →
Configured → Enabled → Running`): el modelo del registro es más rico, pero el que
está en uso es el de la máquina de arriba, sin estados de error explícitos.

## 7. Lo que no está implementado

| Elemento del diseño | Estado | Nota |
|---------------------|--------|------|
| Registro y descubrimiento dinámico de módulos | ❌ | `registerModule()` sin llamadas |
| Estados `InstallError` / `ConfigError` / `RuntimeError` / `UpdateError` | ❌ | Ningún código los asigna |
| `uninstall()` / `stop()` aplicados a componentes reales | ❌ | `stop()` y `disable()` existen en la interfaz, nadie los llama |
| Módulos con `loop()` propio conducidos por `loopAll()` | ❌ | El `loop()` de cada manager se llama desde `SemaCore::loop()` |
| Versionado por módulo (`version()`) | ❌ | Ninguna clase lo implementa; solo existe la firma |
| Driver de `MAX14830` / `SC18IS602B` / `MCP23S17` | ❌ | Solo configuración (ver §4.4) |
| Driver de registros de desplazamiento en builds por defecto | ⚠️ | `SEMA_USE_SHIFT = 0` por defecto; el código existe pero no se aplica |

---

## Ver también

- [Arquitectura](Arquitectura.md) · [Diagramas](Diagramas.md)
- [Tareas-y-concurrencia](Tareas-y-concurrencia.md) · [Rendimiento-y-memoria](Rendimiento-y-memoria.md)
- [Referencia-de-codigo](Referencia-de-codigo.md) · [Referencia-API-interna](Referencia-API-interna.md)
- [Sensores](Sensores.md) · [Buses-y-perifericos](Buses-y-perifericos.md)
