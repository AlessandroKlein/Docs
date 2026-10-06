---
tags:
  - sema
  - desarrollo
---

# Guía de desarrollo (extender SEMA)

> **Tipo:** Guía | **Estado:** Estable | **Firmware:** v1.31.0

Cómo extender SEMA con nuevos sensores, publicadores o reglas.

## Añadir un sensor

1. Crea `include/core/sensors/MiSensor.hpp` + `src/core/sensors/MiSensor.cpp`,
   implementando la interfaz `Sensor`:

```cpp
class MiSensor : public Sensor {
public:
  MiSensor(const char* id, uint8_t sda, uint8_t scl);
  const char* id() const override;
  const char* model() const override;      // p. ej. "BME280"
  const char* interface() const override;  // "I2C" | "1-Wire" | "ADC" | "PCNT" | "UART"
  bool begin() override;
  uint8_t measure(Measurement out[], uint8_t max) override;
  bool healthy() const override;
};
```

2. Regístralo en `SensorFactory::create()`:

```cpp
if (spec.model == "MI_SENSOR") {
  return new MiSensor(spec.id.c_str(), spec.sda, spec.scl);
}
```

3. Añade la librería a `platformio.ini` (si aplica).

## Añadir un publicador

1. Implementa la interfaz `Publisher` (`id()`, `enabled()`, `publish()`).
2. Regístralo en `SemaCore::setup()` y reconfigúralo en `applyPublishers()`.

```cpp
publishers_.registerPublisher(&miPublisher);
```

## Añadir una regla/derivada

- Reglas: se definen por config (`rules[]`) y las evalúa `RuleEngine`.
- Derivadas (p. ej. sensación térmica): extender `DerivedEngine::compute()`.

## Añadir un endpoint REST

1. Declara `void onMiEndpoint();` en `HttpServer.hpp`.
2. Implementa el handler y registra la ruta en `HttpServer::begin()`:

```cpp
server_.on("/api/v1/mi", HTTP_GET, [this]() { onMiEndpoint(); });
```

## Flujo de trabajo (obligatorio)

Tras cada cambio de código:

1. `pio run` (compilar, debe dar SUCCESS).
2. Bump de versión SemVer en `Version.hpp`.
3. Actualizar `CHANGELOG.md`.
4. Commit (Conventional Commits) + `git tag -a vX.Y.Z`.
5. `git push origin main --tags` + `gh release create`.
6. Actualizar `firmware_manifest.json` (SHA-256) y la wiki.

Ver [`CONTINUACION.md`](https://github.com/AlessandroKlein/SEMA/blob/main/docs/CONTINUACION.md)
en el repo SEMA.

## Convenciones

- Usar `nowEpoch()` (de `Time.hpp`) para timestamps (epoch UTC).
- `SEMA_FW_VERSION` sin prefijo `v`; los tags sí usan `vX.Y.Z`.
- Preguntar capacidades (`CapabilityManager`) en lugar de modelo de placa.
