---
tags:
  - sema
  - api
  - codigo
---

# Referencia de API interna

> **Tipo:** API | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Interfaces C++ internas del firmware (`include/core/**`, `include/hw/HwProfile.hpp`).
Las firmas se copiaron literalmente de las cabeceras; cuando un método es `inline`
en el header se indica. Sirve para extender SEMA con sensores, publicadores,
módulos o endpoints sin tocar el núcleo.

Convenciones del código: todo vive en `namespace sema`, con `#pragma once`,
`uint32_t` para milisegundos/epoch, `String` de Arduino para texto y punteros
crudos para los drivers (los `ownedSensors_` de `SemaCore` liberan con `delete`).

## 1. `SemaCore` — orquestador

`include/core/SemaCore.hpp` · implementación `src/core/SemaCore.cpp`.
Singleton con constructor privado; `main.cpp` solo lo llama.

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `instance` | `static SemaCore& instance()` | Instancia única (función `static` local) |
| `setup` | `void setup()` | Inicializa todos los servicios (§4 de [Referencia de código](Referencia-de-codigo.md)) |
| `loop` | `void loop()` | Alimenta el watchdog, atiende WiFi/HTTP/Zigbee/Ethernet, agrega el histórico cada hora y corre el scheduler |
| `applyRules` | `void applyRules()` | Limpia y recarga `rules_`; si `config.rules` está vacío crea la regla por defecto `high_temp` (`EXT`/`temperature` > 40.0) |
| `applyCalibrations` | `void applyCalibrations()` | Limpia y recarga calibraciones; por defecto `-40…85 °C` para `EXT:temperature` e `INT:temperature` |
| `applyGpio` | `void applyGpio()` | `gpio_.apply(config.get().gpio)` |
| `applyPublishers` | `void applyPublishers()` | Reconfigura webhook y MQTT desde `publishers` |
| `applySensors` | `void applySensors()` | Borra y recrea el catálogo (fijo o config-driven) y llama a `beginAll()` |
| `applyShift` | `void applyShift()` | `shift_.apply(...)` solo bajo `#if SEMA_USE_SHIFT` |
| `applyModbus` | `void applyModbus()` | Solo bajo `#if SEMA_USE_MODBUS` |
| `applyCan` | `void applyCan()` | Solo bajo `#if SEMA_USE_CAN` |
| `applyLora` | `void applyLora()` | Solo bajo `#if SEMA_USE_LORA` |
| `applyZigbee` | `void applyZigbee()` | Solo bajo `#if SEMA_USE_ZIGBEE` |
| `applyEthernet` | `void applyEthernet()` | Solo bajo `#if SEMA_USE_ETHERNET` |

Accesores (todos `inline` en el header):

| Accesor | Firma real | Devuelve |
|---------|-----------|----------|
| `modules` | `ModuleRegistry& modules()` | Registro de módulos |
| `events` | `EventBus& events()` | Bus de eventos |
| `config` | `ConfigManager& config()` | Configuración |
| `capabilities` | `CapabilityManager& capabilities()` | `CapabilityManager::instance()` |
| `scheduler` | `Scheduler& scheduler()` | Scheduler |
| `wifi` | `WiFiManager& wifi()` | Gestor WiFi |
| `power` | `PowerManager& power()` | `PowerManager::instance()` |
| `watchdog` | `Watchdog& watchdog()` | TWDT |
| `health` | `HealthMonitor& health()` | Salud |
| `sensors` | `SensorManager& sensors()` | Sensores |
| `history` | `HistoryStore& history()` | Histórico |
| `publishers` | `PublisherManager& publishers()` | Publicadores |
| `rules` | `RuleEngine& rules()` | Reglas |
| `derived` | `DerivedCalculator& derived()` | Derivadas/unidades |
| `eventLog` | `EventLog& eventLog()` | Log de eventos |
| `gpio` | `GpioManager& gpio()` | GPIO |
| `shift` | `ShiftRegisterManager& shift()` | Shift registers |
| `restartCount` | `uint32_t restartCount() const` | Contador `boots` persistido en NVS |
| `modbus` | `ModbusManager& modbus()` | Bajo `#if SEMA_USE_MODBUS` |
| `can` | `CanManager& can()` | Bajo `#if SEMA_USE_CAN` |
| `lora` | `LoraManager& lora()` | Bajo `#if SEMA_USE_LORA` |
| `zigbee` | `ZigbeeManager& zigbee()` | Bajo `#if SEMA_USE_ZIGBEE` |
| `ethernet` | `EthernetManager& ethernet()` | Bajo `#if SEMA_USE_ETHERNET` |
| `detectedDevices` | `const std::vector<DetectedDevice>& detectedDevices() const` | Resultado del escaneo I²C del arranque |

## 2. Ciclo de vida, registro y runtime

### 2.1 `Module` (abstracta) — `include/core/Module.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `id` | `virtual const char* id() const = 0` | Identificador estable (obligatorio) |
| `version` | `virtual const char* version() const = 0` | Versión del módulo (obligatorio) |
| `install` | `virtual bool install()` | Default: `true` |
| `configure` | `virtual bool configure()` | Default: `true` |
| `enable` | `virtual bool enable()` | Default: pone `state_ = ModuleState::Enabled` y devuelve `true` |
| `start` | `virtual void start()` | Default: `state_ = ModuleState::Running` |
| `loop` | `virtual void loop()` | Default: vacío |
| `stop` | `virtual void stop()` | Default: `state_ = ModuleState::Enabled` |
| `disable` | `virtual bool disable()` | Default: `state_ = ModuleState::Disabled` y `true` |
| `state` | `ModuleState state() const` | Estado actual (campo protegido `state_`, default `Available`) |

⚠️ **No existe ninguna clase derivada de `Module` en el repo**: el registro siempre
queda vacío (ver §7 de [Referencia de código](Referencia-de-codigo.md)).

### 2.2 `ModuleRegistry` — `include/core/ModuleRegistry.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `registerModule` | `bool registerModule(Module& module)` | Rechaza (`false`) si el `id()` ya existe |
| `find` | `Module* find(const char* id) const` | Búsqueda por `strcmp`; `nullptr` si no está |
| `enableAll` | `void enableAll()` | Para cada módulo en `Available`: `install` → `configure` → `enable` → `start` |
| `loopAll` | `void loopAll()` | Llama `loop()` solo en `Enabled` o `Running` |
| `count` | `size_t count() const` | Cantidad registrada |

### 2.3 `CapabilityManager` (singleton) — `include/core/CapabilityManager.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `instance` | `static CapabilityManager& instance()` | Instancia única |
| `set` | `void set(Capability c, bool available)` | Prende/apaga el bit `1u << uint8_t(c)` de `mask_` |
| `has` | `bool has(Capability c) const` | Consulta el bit |

### 2.4 `Task` — `include/core/runtime/Task.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| ctor | `Task(TaskFunction fn, const char* name, uint32_t stackBytes, uint32_t priority, void* arg = nullptr)` | `TaskFunction` = `void (*)(void*)` |
| dtor | `~Task()` | Llama a `stop()` |
| copia | `Task(const Task&) = delete` / `Task& operator=(const Task&) = delete` | No copiable |
| `start` | `bool start()` | `xTaskCreate` (afinidad AUTO/`tskNO_AFFINITY`); `false` si ya estaba iniciada |
| `stop` | `void stop()` | `vTaskDelete` y anula el handle |
| `running` | `bool running() const` | `handle_ != nullptr` |

### 2.5 `Scheduler` — `include/core/Scheduler.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `add` | `void add(const char* name, uint32_t intervalMs, std::function<void()> fn)` | Registra una tarea periódica |
| `run` | `void run()` | Ejecuta las vencidas según `millis()`; se llama desde `SemaCore::loop()` |
| `count` | `size_t count() const` | Cantidad de tareas |

`Scheduler::Task` (struct anidado) expone `const char* name`, `uint32_t intervalMs`,
`uint32_t lastRun`, `std::function<void()> fn`.

Tareas realmente registradas en `SemaCore::setup()`: `core.heartbeat` (5 000 ms) y
`sensors.read` (10 000 ms).

### 2.6 `Watchdog` — `include/core/Watchdog.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `begin` | `bool begin(uint32_t timeoutSeconds = 10)` | `esp_task_wdt_init(timeoutSeconds, true)` + `esp_task_wdt_add(NULL)`; acepta `ESP_ERR_INVALID_STATE` si el core Arduino ya lo inicializó |
| `feed` | `void feed()` | `esp_task_wdt_reset()` si arrancó |

### 2.7 `HealthMonitor` — `include/core/HealthMonitor.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `kMaxTasks` | `static constexpr size_t kMaxTasks = 10` | Tope de tareas vigiladas |
| `begin` | `void begin()` | Fija `startMs_` y `lastTickMs_` |
| `tick` | `void tick()` | Marca el heartbeat |
| `setSensorStats` | `void setSensorStats(size_t online, size_t total)` | Alimenta el estado por sensores |
| `registerTask` | `void registerTask(const char* name, uint32_t timeoutMs)` | Alta de tarea; **ignora** si ya hay 10 |
| `taskHeartbeat` | `void taskHeartbeat(const char* name)` | Refresca `lastMs` de esa tarea |
| `taskHealthy` | `bool taskHealthy(const char* name) const` | `(millis() - lastMs) < timeoutMs`; **`true` si la tarea no está registrada** |
| `allTasksHealthy` | `bool allTasksHealthy() const` | `false` si alguna tarea venció |
| `taskCount` | `size_t taskCount() const` | Cantidad registrada |
| `taskName` | `const char* taskName(size_t i) const` | Nombre o `nullptr` si `i` fuera de rango |
| `status` | `const char* status() const` | `"ERROR"` (hay sensores y ninguno responde) → `"DEGRADED"` (alguno caído, heartbeat > 30 s o tarea vencida) → `"HEALTHY"` |
| `uptimeSeconds` | `uint32_t uptimeSeconds() const` | `(millis() - startMs_) / 1000` |

## 3. `EventBus` y helpers de enums

`include/core/EventBus.hpp` · `src/core/EventBus.cpp`.

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `Handler` | `using Handler = std::function<void(const Event&)>` | Tipo de callback |
| `publish` | `void publish(const Event& event)` | **Síncrono**: invoca en línea los handlers cuyo `type` coincide |
| `subscribe` | `void subscribe(EventType type, Handler handler)` | Alta de suscripción (se permiten duplicados) |

Funciones libres `inline` (mismo header):

| Función | Firma real | Descripción |
|---------|-----------|-------------|
| `severityName` | `inline const char* severityName(Severity s)` | `"DEBUG"`, `"INFO"`, `"NOTICE"`, `"WARNING"`, `"ERROR"`, `"CRITICAL"`, `"UNKNOWN"` |
| `eventTypeName` | `inline const char* eventTypeName(EventType t)` | Minúsculas: `"sensor"`, `"rain"`, `"lightning"`, `"battery"`, `"network"`, `"alarm"`, `"system"`, `"wake"`, `"sleep"` |
| `parseEventType` | `inline EventType parseEventType(const char* name)` | Inversa; `nullptr` o desconocido → `EventType::System` |
| `parseSeverity` | `inline Severity parseSeverity(const char* name)` | Inversa (espera mayúsculas); desconocido → `Severity::Info` |

## 4. Configuración — `ConfigManager`

`include/core/ConfigManager.hpp` · `src/core/ConfigManager.cpp`.

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| ctor | `explicit ConfigManager(KeyValueStore& store)` | Recibe el backend clave-valor |
| `load` | `bool load()` | Lee la clave NVS `config`; si no deserializa o no valida, aplica defaults (schema=1, `SEMA-001`, `Estación Norte`, STA, `sema-001`, mDNS, Buenos Aires, INFO, littlefs, 30 días) |
| `save` | `bool save()` | Serializa y guarda; si NVS falla, `store_.clear()` y reintenta **una vez** (se pierde la config anterior) |
| `apply` | `bool apply(const Config& next)` | Valida → guarda backup → persiste; si falla hace rollback y devuelve `false` |
| `get` | `const Config& get() const` | Config vigente |
| `valid` | `bool valid() const` | Bandera de validez |
| `toJson` | `bool toJson(String& out) const` | Alias `inline` de `serialize` |
| `applyJson` | `bool applyJson(const String& json)` | `parseInto` sobre una `Config` temporal + `apply` |
| `saveDashboardLayout` | `bool saveDashboardLayout(const String& layout)` | Clave NVS separada `layout` (no va en el JSON de config) |
| `loadDashboardLayout` | `bool loadDashboardLayout(String& out)` | Lee la clave `layout` |
| `validate` | `bool validate(const Config& c) const` (privado) | Ver §4.1 |
| `serialize` | `bool serialize(String& out) const` (privado) | `DynamicJsonDocument(16384)` con todas las claves |
| `parseInto` | `bool parseInto(const String& in, Config& c)` (privado) | Deserializa con defaults por campo y luego `applyHwProfile(c)` |
| `deserialize` | `bool deserialize(const String& in)` (privado) | `parseInto(in, config_)` |

### 4.1 Reglas de validación reales

| # | Condición | Debe cumplir |
|--:|-----------|--------------|
| 1 | `c.schemaVersion` | `== 1` (cualquier otro valor invalida **toda** la config) |
| 2 | `c.station.id.length()` | `> 0` |
| 3 | `c.network.mode` | `"STA"` o `"AP"` |
| 4 | `c.storage.backend` | `"littlefs"`, `"flash"` o `"sd"` |

## 5. Sensores

### 5.1 `Sensor` (abstracta) — `include/core/sensors/Sensor.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `id` | `virtual const char* id() const = 0` | Id lógico base (p. ej. `"EXT"`) |
| `model` | `virtual const char* model() const = 0` | Modelo físico (p. ej. `"BME280"`) |
| `interface` | `virtual const char* interface() const = 0` | `"I2C"`, `"1-Wire"`, `"ADC"`, `"GPIO"`, `"UART"` |
| `begin` | `virtual bool begin() = 0` | Inicializa y detecta hardware |
| `measure` | `virtual uint8_t measure(Measurement out[], uint8_t max) = 0` | Llena hasta `max` mediciones y devuelve cuántas escribió |
| dtor | `virtual ~Sensor() = default` | Destrucción polimórfica |

### 5.2 `SensorManager` — `include/core/sensors/SensorManager.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `registerSensor` | `void registerSensor(Sensor* sensor)` | Agrega el puntero (no toma propiedad) |
| `clear` | `void clear()` | `sensors_.clear()` (no libera memoria) |
| `beginAll` | `void beginAll()` | Llama `begin()` en todos |
| `readAll` | `void readAll()` | Limpia mediciones; por cada sensor **sano** pide hasta **4** mediciones (`Measurement buffer[4]`), aplica la calibración de `sensorId:channelId` y al final llama a `DerivedEngine::compute()` |
| `setCalibration` | `void setCalibration(const String& key, const Calibration& c)` | Clave `"sensorId:channelId"` |
| `clearCalibrations` | `void clearCalibrations()` | Vacía el mapa |
| `count` | `size_t count() const` | Sensores registrados |
| `onlineCount` | `size_t onlineCount() const` | Los que devuelven `healthy() == true` |
| `describe` | `void describe(std::vector<SensorInfo>& out) const` | `out.clear()` + un `SensorInfo` por sensor |
| `measurements` | `const std::vector<Measurement>& measurements() const` | Última tanda leída |

### 5.3 `SensorFactory` — `include/core/sensors/SensorFactory.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `create` | `static Sensor* create(const SensorSpec& spec)` | `new` del driver según `spec.model`; `nullptr` si el modelo no existe (el llamador debe liberar) |

### 5.4 `I2cScanner` — `include/core/sensors/I2cScanner.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `scan` | `static size_t scan(std::vector<DetectedDevice>& out)` | Recorre 1..126 con `Wire.beginTransmission`; `out.clear()` al empezar |
| `modelForAddress` | `static String modelForAddress(uint8_t address)` | `""` si no lo conoce |

Direcciones mapeadas (verificado en `I2cScanner.cpp`): `0x23` BH1750 · `0x38`/`0x39`
AHT20 · `0x40` SHT31/HTU21D · `0x44`/`0x45` SHT40/SHT3x · `0x5C` AM2320 · `0x61`
SCD30 · `0x62` SCD40/SCD41 · `0x68` MPU6050/DS3231 · `0x76`/`0x77` BME280/BMP280.

### 5.5 Drivers concretos

Todos implementan la misma interfaz de §5.1. La tabla indica el constructor real,
lo que devuelve `model()`/`interface()` y las magnitudes que emite cada `measure()`.

| Clase | Constructor | `model()` / `interface()` | Magnitudes emitidas (`channelId` / `measurement` / `unit`) |
|-------|-------------|---------------------------|-----------------------------------------------------------|
| `AdcSensor` | `AdcSensor(const char* id, uint8_t pin, const char* channel, const char* unit, float scale, float offset)` | `"ESP32-ADC"` / `"ADC"` | `<channel>` / `<channel>` / `<unit>` (valor = lectura × `scale` + `offset`) |
| `Ads1115Sensor` | `Ads1115Sensor(const char* id, uint8_t sda, uint8_t scl, uint8_t channel, const char* name, const char* unit, float scale, float offset)` | `"ADS1115"` / `"I2C"` | `<name>` / `<name>` / `<unit>` (el `channel` es 0..3) |
| `Aht20Sensor` | `Aht20Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"AHT20"` / `"I2C"` | `temperature`/`degC`, `humidity`/`percent` |
| `As3935Sensor` | `As3935Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"AS3935"` / `"I2C"` | `distance` / `lightning_distance` / `km` |
| `Bh1750Sensor` | `Bh1750Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"BH1750"` / `"I2C"` | `light` / `light` / `lux` |
| `Bme280Sensor` | `Bme280Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"BME280"` / `"I2C"` | `temperature`/`degC`, `humidity`/`percent`, `pressure`/`hPa` |
| `Bmp280Sensor` | `Bmp280Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"BMP280"` / `"I2C"` | `temperature`/`degC`, `pressure`/`hPa` |
| `CoSensor` | `CoSensor(const char* id, uint8_t pin, float scale, float offset)` | `"CO"` / `"ADC"` | `co` / `co` / `ppm` |
| `Ds18b20Sensor` | `Ds18b20Sensor(const char* id, uint8_t pin, const char* rom)` | `"DS18B20"` / `"1-Wire"` | `temperature`/`degC` (una por dispositivo; con `rom` vacío recorre el bus) |
| `PcntSensor` | `PcntSensor(const char* id, uint8_t pin, const char* channel, const char* unit, float scale)` | `"PCNT"` / `"GPIO"` | `<channel>` / `<channel>` / `<unit>` (pulsos × `scale`) |
| `Pms5003Sensor` | `Pms5003Sensor(const char* id, uint8_t rxPin, uint8_t txPin)` | `"PMS5003"` / `"UART"` | `pm1`/`pm1`, `pm25`/`pm25`, `pm10`/`pm10`, todas `ug/m3` |
| `Scd30Sensor` | `Scd30Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"SCD30"` / `"I2C"` | `co2`/`ppm`, `temperature`/`degC`, `humidity`/`percent` |
| `Sgp30Sensor` | `Sgp30Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"SGP30"` / `"I2C"` | `eco2`/`ppm`, `tvoc`/`ppb` |
| `Sht31Sensor` | `Sht31Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"SHT31"` / `"I2C"` | `temperature`/`degC`, `humidity`/`percent` |
| `Sht40Sensor` | `Sht40Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"SHT40"` / `"I2C"` | `temperature`/`degC`, `humidity`/`percent` |
| `SolarSensor` | `SolarSensor(const char* id, uint8_t pin, float scale, float offset)` | `"SOLAR"` / `"ADC"` | `solar_radiation` / `solar_radiation` / `W/m2` |
| `Veml6075Sensor` | `Veml6075Sensor(const char* id, uint8_t sda, uint8_t scl)` | `"VEML6075"` / `"I2C"` | `uva`/`W/m2`, `uvb`/`W/m2`, `uvi`/`index` |

`Ds18b20Sensor` es el único driver con destructor propio
(`~Ds18b20Sensor() override`), porque libera `OneWire` y `DallasTemperature`.

## 6. Derivadas y calibración

### 6.1 `DerivedEngine` — `include/core/derived/DerivedEngine.hpp` (motor activo)

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `compute` | `static void compute(std::vector<Measurement>& measurements)` | Busca un par temperatura+humedad del **mismo** `sensorId` y **agrega** 4 mediciones `DERIVED`: `dew_point`, `heat_index`, `vapor_pressure` (hPa) y `absolute_humidity` (g/m3). Si no hay par, no hace nada |
| `dewPoint` | `static float dewPoint(float tempC, float relHum)` | Magnus con `a=17.62`, `b=243.12` (°C) |
| `heatIndex` | `static float heatIndex(float tempC, float relHum)` | Regresión de Rothfusz/NOAA (°C) |
| `saturationVaporPressure` | `static float saturationVaporPressure(float tempC)` | `6.112 · exp(17.67·T/(T+243.5))` en hPa |
| `absoluteHumidity` | `static float absoluteHumidity(float tempC, float relHum)` | `216.7 · e / (T+273.15)` en g/m³ |

### 6.2 `DerivedCalculator` — `include/core/derived/DerivedCalculator.hpp`

Usado por `HttpServer` (histórico y `/api/v1/sensors`), **no** por `SensorManager`.

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `configure` | `void configure(const SystemConfig& sys)` (inline) | Copia la config de sistema (altitud, veleta, resistencias) |
| `compute` | `void compute(const std::vector<Measurement>& raw, std::vector<Measurement>& out, const String& units)` | Emite lo calculable con las entradas válidas: `dew_point`, `heat_index`, `vpd` (kPa), `wind_chill`, `qnh` (hPa), `barometric_altitude` (m), `aqi` (index), `rain_rate` (mm/h), `rain_accumulated`, y `wind_direction` si `windDirectionPin != 0` (lee el ADC) |
| `convertUnit` | `static float convertUnit(float value, const String& measurement, const String& unit, bool imperial, String& outUnit)` | A imperial: `degF`, `inHg` (÷33.8639), `mph` (×2.23694), `in` (÷25.4), `in/h`, `ft` (×3.28084). Con `imperial=false` devuelve el valor y `outUnit = unit` |
| `windVaneRawAngle` | `static float windVaneRawAngle(uint16_t adc, const SystemConfig& sys)` | Nearest-neighbour sobre 16 posiciones: 8 resistencias directas + 8 en paralelo, ángulos cada 22.5° |
| `windDirection` | `static float windDirection(uint16_t adc, const SystemConfig& sys)` | Ángulo bruto − `windNorthOffset`, normalizado a [0, 360) |

Tabla AQI usada por `aqiFromPm25()` (privada): 0–12→0–50 · 12.1–35.4→51–100 ·
35.5–55.4→101–150 · 55.5–150.4→151–200 · 150.5–250.4→201–300 · 250.5–350.4→301–400 ·
350.5–500.4→401–500; por encima, 500.

### 6.3 Calibración — `include/core/Calibration.hpp`

| Elemento | Firma real | Descripción |
|----------|-----------|-------------|
| `Calibration` | struct: `float offset = 0.0f`, `float gain = 1.0f`, `float min = 0.0f`, `float max = 0.0f`, `bool hasRange = false`, `bool enabled = false` | Parámetros por canal |
| `applyCalibration` | `void applyCalibration(Measurement& m, const Calibration& c)` | Si `enabled`: `value = value*gain + offset`; si `hasRange` y queda fuera de `[min,max]`, marca `Quality::OutOfRange` |

## 7. Alarmas — `RuleEngine`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| ctor | `explicit RuleEngine(EventBus& bus)` | Necesita el bus para publicar |
| `addRule` | `void addRule(const Rule& rule)` | Agrega al vector (no deduplica) |
| `clear` | `void clear()` | `rules_.clear()` |
| `evaluate` | `void evaluate(const std::vector<Measurement>& measurements)` | Para cada regla × medición que matchea publica un `Event` `Alarm` con severidad `Warning`, `source = m.sensorId`, `correlationId = r.id` y `value = (int32_t)(m.value * 100.0f)` |

`matches()` (privado): si `rule.sensorId` no está vacío debe coincidir; `m.channelId`
debe ser igual a `rule.channelId`; luego aplica `Gt`/`Lt`/`Ge`/`Le`. **No** hay
histéresis, ni duración mínima, ni acciones: cada evaluación que cumple vuelve a
emitir el evento.

`parseRuleOp(const char* name)` (inline en `alarms/Rule.hpp`): `"lt"`, `"ge"`,
`"le"`; cualquier otro valor (incluido `"gt"`) → `RuleOp::Gt`.

## 8. Almacenamiento

### 8.1 `KeyValueStore` (abstracta) — `include/core/storage/Storage.hpp`

| Método | Firma real |
|--------|-----------|
| `begin` | `virtual bool begin(const char* name) = 0` |
| `getString` | `virtual bool getString(const char* key, String& out) = 0` |
| `putString` | `virtual bool putString(const char* key, const char* value) = 0` |
| `getUInt` | `virtual bool getUInt(const char* key, uint32_t& out) = 0` |
| `putUInt` | `virtual bool putUInt(const char* key, uint32_t value) = 0` |
| `clear` | `virtual bool clear() = 0` |

### 8.2 `NvsStore` — `include/core/storage/NvsStore.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| ctor / dtor | `NvsStore()` · `~NvsStore() override` | Crea/libera la `Preferences` interna |
| `begin` | `bool begin(const char* name) override` | `prefs_->begin(name, false)` (solo lectura-escritura, sin borrar) |
| `getString` / `getUInt` | `bool getString(const char* key, String& out) override` · `bool getUInt(const char* key, uint32_t& out) override` | Devuelven `false` si la clave no existe (`isKey`) |
| `putString` / `putUInt` | `bool putString(const char* key, const char* value) override` · `bool putUInt(const char* key, uint32_t value) override` | `true` si escribió > 0 bytes |
| `clear` | `bool clear() override` | Borra el namespace |

### 8.3 `HistoryStore` — `include/core/storage/HistoryStore.hpp`

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `begin` | `bool begin(const char* path = "/history.jsonl")` | Fija `path_` y `aggPath_ = path + ".agg"`; `count_ = 0`; **no** monta nada |
| `enableSd` | `bool enableSd(uint8_t csPin)` | `SD.begin(csPin)`; si monta, cuenta las líneas existentes. **Sin SD no hay histórico** |
| `sdEnabled` | `bool sdEnabled() const` | Estado del montaje |
| `append` | `bool append(const Measurement& m)` | `false` sin SD; rota si llegó a `maxEntries_`; escribe una línea JSON con claves `ts`, `sensor`, `channel`, `measurement`, `value`, `unit`, `quality`, `seq` |
| `readRecent` | `bool readRecent(std::deque<Measurement>& out, size_t maxCount)` | Lee el archivo completo y conserva las últimas `maxCount` |
| `aggregate` | `bool aggregate(uint32_t bucketSeconds, uint32_t cutoffEpoch)` | Lo anterior a `cutoffEpoch` se promedia por bucket (clave `sensorId｜measurement｜bucket`) y se anexa al `.agg`; el raw se reescribe solo con lo reciente |
| `readAggregated` | `bool readAggregated(std::deque<Measurement>& out, size_t maxCount)` | Lee el archivo `.agg` |
| `count` | `size_t count() const` | Entradas contadas |
| `maxEntries` | `uint32_t maxEntries() const` | Límite actual (default **10 000**) |
| `setMaxEntries` | `void setMaxEntries(uint32_t max)` (inline) | Cambia el límite |
| `setRetentionSeconds` | `void setRetentionSeconds(uint32_t s)` (inline) | `0` = sin retención |
| `prune` | `bool prune(uint32_t nowEpoch)` | Descarta lo más viejo que `nowEpoch - retentionSeconds_`; no hace nada sin SD o con retención 0 |
| `rotate` | `bool rotate()` (privado) | Conserva la **mitad más reciente** reescribiendo el archivo |

En `SemaCore::setup()` la retención se fija a `storage.retentionDays * 86400` y
`SemaCore::loop()` llama a `aggregate(3600, now - retentionDays*86400)` cada hora.
**`prune()` no se invoca desde el núcleo** (la poda efectiva la hace `aggregate()`) y
`setMaxEntries()` tampoco: el límite queda en los 10 000 por defecto.
`readAggregated()` sí se usa, desde `GET /api/v1/history` (`HttpServer.cpp:2329`).

## 9. Eventos persistentes — `EventLog`

`include/core/events/EventLog.hpp`.

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| ctor | `explicit EventLog(EventBus& bus, size_t maxEntries = 100)` | Se suscribe a **todos** los `EventType` iterando `0..Sleep` (con un `TODO` para usar un centinela `Count`) |
| `begin` | `void begin(const char* path = "/events.jsonl")` | `LittleFS.begin(true)`, lee el archivo existente y carga hasta `maxEntries_` eventos |
| `events` | `const std::deque<Event>& events() const` | Buffer en RAM |
| `onEvent` | `void onEvent(const Event& e)` (privado) | Encola (descarta el más viejo al pasar el límite), serializa a JSON con claves `ts`, `type`, `source`, `rule`, `severity`, `value` y anexa al archivo |
| `rotate` | `bool rotate()` (privado) | Cuando `fileCount_ >= maxEntries_ * 2` conserva las últimas `maxEntries_` líneas |
| `parseEvent` | `bool parseEvent(const String& line, Event& e)` (privado) | Inverso de la serialización |

Ojo: `EventLog` guarda `timestampMs` (millis) en la clave `ts`, mientras que
`HistoryStore` guarda epoch en su propio `ts`.

## 10. Publicadores

| Clase | Método | Firma real | Descripción |
|-------|--------|-----------|-------------|
| `Publisher` | `id` | `virtual const char* id() const = 0` | Identificador |
| `Publisher` | `enabled` | `virtual bool enabled() const = 0` | `false` → no se publica |
| `Publisher` | `publish` | `virtual bool publish(const Measurement& m) = 0` | Publica una medición |
| `PublisherManager` | `registerPublisher` | `void registerPublisher(Publisher* publisher)` | Alta (no toma propiedad) |
| `PublisherManager` | `publishAll` | `void publishAll(const Measurement& m)` | Recorre y publica solo los `enabled()` |
| `PublisherManager` | `count` | `size_t count() const` | Cantidad registrada |
| `MqttPublisher` | ctor | `MqttPublisher(const char* id, const char* host, uint16_t port, const char* topic)` | — |
| `MqttPublisher` | `configure` | `void configure(const char* host, uint16_t port, const char* topic, const char* user, const char* pass)` | Reconfigura en caliente |
| `MqttPublisher` | `enabled` | `bool enabled() const override` | `host_.length() > 0` |
| `MqttPublisher` | `publish` | `bool publish(const Measurement& m) override` | Conecta si hace falta (con usuario/clave si hay) y publica el JSON canónico en `topic_` |
| `HttpPublisher` | ctor | `HttpPublisher(const char* id, const char* url)` | — |
| `HttpPublisher` | `setUrl` | `void setUrl(const char* url)` (inline) | Cambia el webhook |
| `HttpPublisher` | `enabled` | `bool enabled() const override` | `url_.length() > 0` |
| `HttpPublisher` | `publish` | `bool publish(const Measurement& m) override` | `POST` con `Content-Type: application/json` y **timeout 2 000 ms**; `true` si `0 < código < 400` |

Claves del JSON publicado (idénticas en MQTT y HTTP, `DynamicJsonDocument(256)`):
`station_id`, `sensor_id`, `channel_id`, `measurement`, `value`, `unit`, `quality`,
`sequence`, `timestamp`.

## 11. Red y buses

| Clase | Método | Firma real | Descripción |
|-------|--------|-----------|-------------|
| `WiFiManager` | `begin` | `void begin(const String& mode, const String& ssid, const String& password, const String& hostname, const String& ip, const String& gateway, const String& subnet, const String& dns, bool mdns)` | `"STA"` con SSID no vacío → `WiFi.begin` (IP estática si `ip`, `gateway`, `subnet` y `dns` parsean); en cualquier otro caso levanta AP con el `hostname` |
| `WiFiManager` | `loop` | `void loop()` | Reconexión con backoff exponencial 2 s → … → 60 s; lo reinicia al reconectar |
| `WiFiManager` | `connected` | `bool connected() const` | En AP siempre `true` |
| `WiFiManager` | `localIP` | `String localIP() const` | `localIP()` en STA, `softAPIP()` en AP |
| `WiFiManager` | `rssi` | `int32_t rssi() const` | `0` en AP |
| `WiFiManager` | `isAp` | `bool isAp() const` (inline) | `!staMode_` |
| `WiFiManager` | `mdnsStarted` | `bool mdnsStarted() const` (inline) | mDNS arrancado (nombre saneado: solo alfanuméricos y guiones) |
| `EthernetManager` | `apply` | `void apply(const EthernetConfig& cfg)` | LAN8720A RMII (`SEMA_NATIVE_ETH`) o W5500 por SPI + `esp_eth`; deja `ready_` |
| `EthernetManager` | `loop` | `void loop()` | **Vacío a propósito**: lwIP gestiona el DHCP |
| `EthernetManager` | `connected` | `bool connected() const` | Nativo: `ETH.linkUp()`; W5500: IP ≠ 0 |
| `EthernetManager` | `localIP` | `String localIP() const` | IP actual o `String()` vacío |
| `EthernetManager` | `enabled` / `config` | `bool enabled() const` (inline) · `const EthernetConfig& config() const` (inline) | Estado de la config |
| `ModbusManager` | `apply` | `void apply(const ModbusConfig& cfg)` | `Serial2.begin(baud, SERIAL_8N1, rx, tx)` + `ModbusMaster::begin(slaveId, Serial2)`; DE/RE con `preTransmission`/`postTransmission` |
| `ModbusManager` | `read` | `uint8_t read()` | `readHoldingRegisters(registerAddr, registerCount)`; llena `values_` solo si `ku8MBSuccess` (0x00). `0xFF` si no está listo |
| `ModbusManager` | `ready` / `config` / `values` | `bool ready() const` · `const ModbusConfig& config() const` · `const std::vector<uint16_t>& values() const` | Estado, config y últimos registros |
| `CanManager` | `apply` | `void apply(const CanConfig& cfg)` | `TWAI_MODE_NORMAL` + timing por velocidad (`125000`/`250000`/`500000`/`1000000`; default 500 k) y filtro accept-all |
| `CanManager` | `send` | `bool send(uint32_t id, const uint8_t* data, uint8_t dlc, bool extd)` | `false` si no listo o `dlc > 8`; espera hasta 1 000 ms |
| `CanManager` | `receive` | `bool receive(uint32_t& id, uint8_t* data, uint8_t& dlc, bool& extd)` | No bloqueante (timeout 0) |
| `CanManager` | `ready` / `config` | `bool ready() const` (inline) · `const CanConfig& config() const` (inline) | Estado y config |
| `LoraManager` | `apply` | `void apply(const LoraConfig& cfg)` | Recrea `Module`+`SX1262` (RadioLib), `begin(freq, bw, sf, cr, SYNC_WORD_PRIVATE, txPower)` y queda en `startReceive()` |
| `LoraManager` | `send` | `bool send(const uint8_t* data, uint8_t len)` | `transmit()` y vuelve a modo recepción |
| `LoraManager` | `receive` | `uint8_t receive(uint8_t* data, uint8_t maxLen)` | Bytes leídos, `0` si no hay nada |
| `LoraManager` | `ready` / `config` | `bool ready() const` (inline) · `const LoraConfig& config() const` (inline) | Estado y config |
| `ZigbeeManager` | `apply` | `void apply(const ZigbeeConfig& cfg)` | `Serial1.begin(baud, SERIAL_8N1, rx, tx)` y envía `SYS_RESET` (`0x4100`) |
| `ZigbeeManager` | `loop` | `void loop()` | Parseo incremental de tramas ZNP: `SOF(0xFE) LEN CMD0 CMD1 payload FCS` |
| `ZigbeeManager` | `send` | `bool send(uint16_t destination, const uint8_t* data, uint8_t len)` | `AF_DATA_REQUEST` (`0x2401`); rechaza `len == 0` o `len > 110` |
| `ZigbeeManager` | `available` / `lastSrc` | `bool available() const` (inline) · `uint16_t lastSrc() const` (inline) | ¿Hay mensaje pendiente? / dirección origen |
| `ZigbeeManager` | `takeMessage` | `void takeMessage(uint8_t* out, uint8_t maxLen, uint8_t& len)` | Copia y limpia la bandera de mensaje (buffer interno de 128 B) |

## 12. GPIO y expansores

| Clase | Método | Firma real | Descripción |
|-------|--------|-----------|-------------|
| `GpioManager` | `apply` | `void apply(const std::vector<GpioSpec>& specs)` | Reinicia el MCP23017; si algún spec tiene `expanderAddr != 0` hace `mcp_.begin_I2C(addr)` con el **primero** que encuentre. `"output"`/`"input"`/`"input_pullup"`/`"input_pulldown"` → `OUTPUT`/`INPUT`/`INPUT_PULLUP`/`INPUT_PULLDOWN`; en `"output"` aplica `initial` |
| `GpioManager` | `read` | `int read(uint8_t pin) const` | Por MCP23017 si el spec lo indica y está listo; si no, `digitalRead` |
| `GpioManager` | `write` | `void write(uint8_t pin, int value)` | Idem con `digitalWrite`/`mcp_.digitalWrite` |
| `GpioManager` | `specs` | `const std::vector<GpioSpec>& specs() const` (inline) | Specs aplicados |
| `ShiftRegisterManager` | `apply` | `void apply(const std::vector<ShiftRegisterConfig>& cfg)` | Configura `SEMA_SPI_MOSI` (OUTPUT, o INPUT para 74HC165), `SEMA_SPI_SCK` y cada `latchPin`; ignora `latchPin == 0` |
| `ShiftRegisterManager` | `writeByte` | `void writeByte(uint8_t value)` | `shiftOut(MSBFIRST)` a **todos** los 74HC595 (latch LOW → shift → HIGH) |
| `ShiftRegisterManager` | `readByte` | `uint8_t readByte()` | `shiftIn(MSBFIRST)` del **primer** 74HC165 (pulso de carga de 5 µs); `0` si no hay |
| `ShiftRegisterManager` | `configured` / `hasOutput` / `hasInput` | `bool configured() const` (inline) · `bool hasOutput() const` · `bool hasInput() const` | Estado de la config |

## 13. Energía — `PowerManager` (singleton)

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `instance` | `static PowerManager& instance()` | Instancia única |
| `profile` | `EnergyProfile profile() const` (inline) | Perfil actual (default `Normal`) |
| `setProfile` | `void setProfile(EnergyProfile p)` (inline) | **Solo cambia la variable**: ningún driver consume el perfil todavía |
| `sleep` | `void sleep(uint64_t seconds)` | `esp_sleep_enable_timer_wakeup(seconds * 1000000ULL)` + `esp_deep_sleep_start()` (el parámetro es **segundos**) |
| `enableRainWakeup` | `void enableRainWakeup(uint8_t pin)` | `esp_sleep_enable_ext0_wakeup(pin, 1)` (nivel HIGH; requiere pin RTC) |
| `wakeReason` | `uint32_t wakeReason() const` | `static_cast<uint32_t>(esp_sleep_get_wakeup_cause())` |

⚠️ `sleep()` y `setProfile()` no se invocan desde `SemaCore`: hoy el firmware no
entra en deep sleep por sí solo.

## 14. `HttpServer`

`include/core/web/HttpServer.hpp` · `src/core/web/HttpServer.cpp` (2 637 líneas).

| Método | Firma real | Descripción |
|--------|-----------|-------------|
| `begin` | `void begin(SemaCore& core)` | Genera el token de sesión (16 hex de `esp_random`), registra las cabeceras `X-API-Key` y `Cookie`, `server_.begin()` y `ws_.begin()` |
| `loop` | `void loop()` | `server_.handleClient()` + `ws_.loop()` |
| `broadcastMeasurements` | `void broadcastMeasurements(const std::vector<Measurement>& measurements)` | Si hay clientes WS, emite `{"type":"measurements","data":[…]}` por el puerto 81 (sin `station_id` ni `timestamp`) |
| `authorized` | `bool authorized()` (privado) | Acepta `X-API-Key` contra `apiKey`, `serverKey` o los valores del JSON `extraKeys`; **sin ninguna clave configurada devuelve `true`** |
| `webAuthed` | `bool webAuthed()` (privado) | Sin login configurado, o sesión válida, o clave API |
| `sessionAuthorized` | `bool sessionAuthorized()` (privado) | Sesión por cookie `sema_auth` con timeout deslizante de 1 h (`kSessionTimeoutMs = 3600000`) |

Handlers privados registrados (`HttpServer.cpp` llama a `server_.on(...)` **53 veces**
entre las líneas 52 y 121; los dos `onNotFound`/assets se cuentan aparte):

| Grupo | Handlers |
|-------|----------|
| Páginas | `onRoot`, `onNetworkPage`, `onSecurityPage`, `onSystemPage`, `onWindPage`, `onSensorsPage`, `onSensorsViewPage`, `onEventsPage`, `onLoginPage`, `onLogout`, `onNotFound` |
| API de estado | `onStatus`, `onHealth`, `onSystem`, `onCapabilities`, `onNetwork`, `onEnergy`, `onDiagnostics`, `onSensors` |
| Configuración | `onConfig`, `onConfigPut`, `onConfigNetwork`, `onConfigSystem`, `onConfigSensors`, `onConfigIo`, `onConfigBuses`, `onApiKeys`, `onWindNorth`, `onWindResistors`, `onDashboardLayout`, `onDashboardLayoutGet` |
| Datos | `onHistory`, `onEvents`, `onAlarms`, `onWifiScan` |
| Almacenamiento/OTA | `onBackup`, `onRestart`, `onOta`, `onOtaUpload`, `onUpdateCheck` |
| Seguridad | `onLoginPost` |
| GPIO/expansores | `onGpio`, `onGpioWrite`, `onShift`, `onShiftWrite` |
| Buses (condicionales) | `onModbus` (`SEMA_USE_MODBUS`), `onCan`/`onCanWrite` (`SEMA_USE_CAN`), `onLora`/`onLoraWrite` (`SEMA_USE_LORA`), `onZigbee`/`onZigbeeWrite` (`SEMA_USE_ZIGBEE`) |

Estado interno relevante para extenderlo: `bool otaAuthorized_`,
`String otaExpectedSha_`, `bool otaShaOk_`, `String sessionToken_`,
`uint32_t failedLogins_` (bloqueo de 60 s a los 5 fallos), `uint32_t lockoutUntilMs_`
y `uint32_t sessionStartMs_` (`0` = sin sesión).

---

## Ver también

- [Referencia de código](Referencia-de-codigo.md) · [Enumeraciones y tipos](Enumeraciones-y-tipos.md) · [Arquitectura](Arquitectura.md) · [Módulos y ciclo de vida](Modulos-y-ciclo-de-vida.md) · [Tareas y concurrencia](Tareas-y-concurrencia.md) · [Diagnóstico y salud](Diagnostico-y-salud.md) · [API REST](API-REST.md) · [Referencia de configuración](Referencia-configuracion.md) · [Pruebas y validación](Pruebas-y-validacion.md) · [Compatibilidad de versiones](Compatibilidad-de-versiones.md)
