---
tags:
  - invernadero
  - api
  - referencia
---

# Referencia de la API interna (clases)

> **Tipo:** Referencia técnica | **Estado:** Estable | **Fecha:** 2026-10-02

API pública de cada clase del firmware (métodos, parámetros y qué hacen).
Fuente: `include/**/*.hpp`.

## `config/ConfigManager`

| Método | Descripción |
|--------|-------------|
| `void begin()` | Abre NVS y carga (o crea) la configuración |
| `SystemConfig get() const` | Copia protegida por mutex |
| `void set(const SystemConfig&)` | Actualiza, versiona y persiste (guardando la anterior para rollback) |
| `void save()` | Persiste la configuración actual y la anterior |
| `void factoryReset()` | Restablece todo a fábrica |
| `void resetNetwork()` | Resetea solo parámetros de red |
| `void resetAutomation()` | Resetea solo control/reglas |
| `bool rollback()` | Restaura la configuración anterior |
| `bool auth(user, pass) const` | Valida credenciales de admin |
| `String adminUser() const` | Usuario administrador |
| `String apiToken() const` | Token de API vigente |
| `String rotateApiToken()` | Genera y persiste uno nuevo aleatorio |
| `void revokeApiToken()` | Elimina el token |
| `bool validateApiToken(t) const` | Verifica un token |
| `static String toJson(cfg)` / `static bool fromJson(json, out)` | Serialización |
| `static void migrate(cfg)` | Migra esquemas antiguos |
| `static String mergeLayerJson(base, layer)` | Merge profundo de capas |
| `void ensureAdminPassword()` | Deriva la clave del UID si está vacía |

## `sensors/SensorManager`

| Método | Descripción |
|--------|-------------|
| `void begin(cfg, pins, SensorRegistry*)` | Inicializa buses y drivers; direcciones I²C del catálogo |
| `void reconfigure(cfg)` | Aplica cambios de configuración |
| `void update()` | Lee todos los sensores habilitados |
| `float temperature()/humidity()/soilMoisture(z)/lightLux()/co2()` | Getters tipados |
| `float flowRate()/flowAccumulated()/tankLevel()/rainAccum()/windSpeed()/ph()/ec()` | Getters tipados |
| `bool floatLow()/floatHigh()` | Flotadores de seguridad |
| `float vpd()/dewPoint()` | Variables calculadas |
| `SensorStatus statusOf(idx)` | Calidad de un slot |
| `void snapshot(out, max, n)` / `String toJson()` | Estado para API |
| `uint8_t scanModbus(found, max)` | Escaneo RS485 |
| `ModbusStats modbusStats()` | Estadísticas del bus |
| `ModbusRtu* modbusRtu()` | Driver compartido para el gateway |

## `actuators/ActuatorManager`

| Método | Descripción |
|--------|-------------|
| `void begin(cfg, ShiftRegister595*, Mcp23017* pool, uint8_t n)` | Construye los slots desde la tabla de roles |
| `void setRequest(role, index, pct)` | Orden deseada (0..100) |
| `void setSafetyOverride(role, index, pct)` | Sobrescritura de seguridad |
| `void clearSafetyOverrides()` | Limpia sobrescrituras |
| `void apply()` | Resuelve seguridad + orden y escribe al hardware |
| `void allSafeState()` | Apaga todo (arranque) |
| `float output(role, index)` / `bool isOn(role, index)` | Estado |
| `void snapshot(out, max, n)` / `toJson()` | Estado para API |
| `setPump/setValve/setFan/setExtractor/setLight/...` | Setters por rol |

## `hardware/BusManager`

`begin(const PinConfig&)` · `registerBus(type, index, enabled)` · `claim(type, index, owner)` ·
`release(...)` · `isRegistered/isBusy/owner/state(...)` · `scanI2c(found, max, index)` ·
`count()` · `toJson()`

## `hardware/HardwareManager`

`begin(const PinConfig&)` · `registerNode(node)` · `setNodeEnabled(id, en)` ·
`getNode(id, out)` · `hasNode(id)` · `nodeCount()` · `scanI2c(...)` · `detectI2cJson()` ·
`snapshot(...)` · `toJson()` · `fromJson(json)` · `load()` · `save()`

## `hardware/SpiManager`

`bool begin(sck, miso, mosi, freq=4MHz)` · `void end()` · `SPIClass& bus()` ·
`bool isReady()` · `select(cs)` · `deselect(cs)` · `beginTransaction()` · `endTransaction()`

## `hardware/AdcManager`

`void begin(SpiManager*, cs, AdcKind, channels=8)` · `bool available()` · `AdcKind kind()` ·
`uint8_t channels()` · `uint16_t readRaw(ch)` · `float readVoltage(ch, vref=3.3)` ·
`uint16_t maxRaw()`

## `hardware/Mcp23017` (I²C) y `Mcp23s17` (SPI)

| MCP23017 | MCP23S17 |
|----------|----------|
| `begin(addr, TwoWire*=&Wire)` | `begin(SpiManager*, cs, addr=0)` |
| `pinMode(pin, mode)` | `pinMode(pin, mode)` |
| `digitalWrite(pin, val)` | `digitalWrite(pin, val)` |
| `digitalRead(pin)` | `digitalRead(pin)` |
| `setPullup(pin, en)` | `setPullup(pin, en)` |
| `writePort(port, val)` / `readPort(port)` | idem |

## `hardware/ShiftRegister595` y `ShiftRegister165`

**595 (salidas):** `begin(mosi, sclk, latch, count)` · `allOff()` ·
`setChannelPercent(ch, pct)` · `setPwmEnabled(en, freq, bits)` · `commit()`

**165 (entradas):** `begin(data, clock, latch, numChips=1)` · `uint32_t read()` ·
`readByte(chip)` · `readBit(bit)` · `numInputs()`

## `hardware/ModbusRtu`

`begin(serial, dePin, baud=9600)` · `readHoldingRegisters(slave, addr, count, out, timeout=200)` ·
`readInputRegisters(...)` · `scan(found, max)` · `ModbusStats stats()`

## `sensors/ModbusGateway`

`begin(ModbusRtu*, ModbusProfileRegistry*)` · `rebuild()` · `tick()` ·
`static float convert(regs, tipo, scale, offset)` · `count()` · `snapshot(...)` · `toJson()`

## `sensors/SensorRegistry`

`registerSensor(e)` · `buildFromConfig(cfg)` · `remove(id)` · `get(id, out)` · `has(id)` ·
`count()` · `countEnabled()` · `snapshot(...)` · `toJson()` · `fromJson(json)` ·
`load()` · `save()`

## `network/NetworkManager`

`begin(cfg)` · `loop()` · `connected()` · `ip()` · `isApMode()` · `isEthernet()` ·
`Client* client()` · `startAp/startSta/startEthernet(cfg)`

## `network/MqttManager`

`begin(cfg, Client*, bool(*netUp)())` · `loop()` · `enabled()` · `connected()` ·
`publishSensors/Status/Actuators/Weather(json)` · `onMessage(...)` ·
`consumeCommand(out)`

## `system/Device`

`static uid()` · `static info(cfg)` · `static resetCause()` · `static setState(s)` ·
`static state()` · `static deviceJson(cfg)` · `static capabilitiesJson(cfg)`

## `storage/StorageManager`

`begin(formatOnFail)` · `beginSD(csPin)` · `mounted()` · `backend()` ·
`writeFile/appendFile/readFile/exists/remove(path)` · `listDir(...)` ·
`usedBytes()/totalBytes()` · `toJson()`

## `control/RuleEngine`

`update()` · `addRule(r)` · `removeRule(index)` · `clear()` · `count()` ·
`snapshot(...)` · `toJson()` · `float readVariable(v, zone)` ·
`static bool evaluate(op, value, threshold)`

## `api/WebSocketServer`

`begin(ConfigManager*, SensorManager*, ActuatorManager*)` · `loop()` ·
`broadcastState()`

## `core/PinConfig`

`PinConfig defaultPinConfig()` · `String pinConfigToJson(p)` · `bool pinConfigFromJson(json, out)` ·
`PinConfigManager::begin()/get()/set(p)/reset()`

## `core/EventBus`, `core/Scheduler`

`EventBus`: publicar/consumir eventos tipados entre tareas.
`Scheduler`: tareas periódicas registradas por capacidades.

Ver también: [Referencia de código](Referencia-de-codigo.md) ·
[Enumeraciones y tipos](Enumeraciones-y-tipos.md) · [API REST](API-REST.md).
