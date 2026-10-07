---
tags:
  - sema
  - api
  - codigo
---

# Referencia de API interna

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.35.0

Interfaces C++ internas (`include/core/`). Útiles para extender SEMA con módulos,
sensores o publicadores.

## `SemaCore` (singleton)

```cpp
static SemaCore& instance();
void setup();
void loop();
void applySensors();       // re-crea el catálogo de sensores
void applyRules();         // re-aplica reglas
void applyCalibrations();  // re-aplica calibración
void applyGpio();          // re-aplica GPIO
void applyPublishers();    // re-aplica webhook/MQTT
```

Accesores:

```cpp
modules() events() config() capabilities() scheduler() wifi() power()
watchdog() health() sensors() history() publishers() rules() eventLog()
gpio() detectedDevices()
```

## `EventBus`

```cpp
using Handler = std::function<void(const Event&)>;
void publish(const Event& event);
void subscribe(EventType type, Handler handler);
```

## `Sensor` (interfaz de driver)

```cpp
class Sensor {
public:
  virtual const char* id() const = 0;
  virtual const char* model() const = 0;
  virtual const char* interface() const = 0;  // "I2C", "1-Wire", "ADC", "PCNT", "UART"
  virtual bool begin() = 0;
  virtual uint8_t measure(Measurement out[], uint8_t max) = 0;
  virtual bool healthy() const = 0;
};
```

## `SensorManager`

```cpp
void registerSensor(Sensor* sensor);
void clear();
void beginAll();
void readAll();
void setCalibration(const String& key, const Calibration& c);
void clearCalibrations();
size_t count() const;
size_t onlineCount() const;
void describe(std::vector<SensorInfo>& out) const;
const std::vector<Measurement>& measurements() const;
```

## `SensorFactory`

```cpp
static Sensor* create(const SensorSpec& spec);  // nullptr si model desconocido
```

## `RuleEngine`

```cpp
void addRule(const Rule& rule);
void clear();
void evaluate(const std::vector<Measurement>& measurements);
```

## `HistoryStore`

```cpp
void begin();
void append(const Measurement& m);
void readRecent(std::deque<Measurement>& out, size_t maxCount);
size_t count() const;
size_t maxEntries() const;
```

## `EventLog`

```cpp
void begin(const char* path = "/events.jsonl");
const std::deque<Event>& events() const;
```

## `Publisher` / `PublisherManager`

```cpp
class Publisher {
public:
  virtual const char* id() const = 0;
  virtual bool enabled() const = 0;
  virtual bool publish(const Measurement& m) = 0;
};

class PublisherManager {
public:
  void registerPublisher(Publisher* publisher);
  void publishAll(const Measurement& m);
};
```

- `HttpPublisher::setUrl(const char* url)` — actualiza el webhook.
- `MqttPublisher::configure(host, port, topic, user, pass)` — actualiza MQTT.

## `GpioManager`

```cpp
void apply(const std::vector<GpioSpec>& specs);
int read(uint8_t pin) const;
void write(uint8_t pin, int value);
const std::vector<GpioSpec>& specs() const;
```

## `WiFiManager`

```cpp
bool connected() const;
String localIP() const;
int rssi() const;
bool isAp() const;
bool mdnsStarted() const;
void loop();
```

## `PowerManager` (singleton)

```cpp
void sleep(uint64_t us);
void enableRainWakeup(uint8_t pin);
uint32_t wakeReason() const;
EnergyProfile profile() const;
```

## `HealthMonitor`

```cpp
void begin();
void tick();
const char* status() const;      // "HEALTHY" | "DEGRADED" | "ERROR"
uint32_t uptimeSeconds() const;
```

## `Watchdog`

```cpp
void begin(uint32_t timeoutSeconds);
void feed();
```

## `DerivedEngine`

```cpp
void compute(const Measurement& in, std::vector<Measurement>& out);
// dew point, heat index, vapor pressure, absolute humidity
```

## `ConfigManager`

```cpp
bool load(); bool save();
bool apply(const Config& next);           // valida + persiste + rollback
const Config& get() const;
bool toJson(String& out) const;
bool applyJson(const String& json);
```
