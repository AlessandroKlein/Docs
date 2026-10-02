---
tags:
  - invernadero
  - desarrollo
---

# Guía de desarrollo (cómo extender)

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-02

Cómo agregar drivers, expansores, endpoints y campos de configuración sin romper
la arquitectura.

## 1. Entorno

```bash
pip install platformio          # o usar la extensión de VS Code
pio run                         # compilar (SUCCESS esperado)
pio run -t upload               # flashear
pio device monitor              # monitor serie (115200)
```

Librerías externas declaradas en `platformio.ini`
(`ArduinoJson`, `PubSubClient`, `Ethernet`, …).

## 2. Convenciones del código

- **Una clase por archivo**: `.hpp` en `include/<modulo>/`, `.cpp` en `src/<modulo>/`.
- Namespace único: `gh::`.
- Comentar el **por qué**, no el **qué**.
- Verificar `pio run` **antes** de commitear (regla de trabajo).
- No hardcodear pines ni secretos: usar `PinConfig` / NVS.

## 3. Agregar un sensor

1. Crear `include/sensors/MiSensor.hpp` + `src/sensors/MiSensor.cpp`.
2. Agregar el enumerador en `SensorType` (`core/Types.hpp`).
3. **Instanciarlo** en `SensorManager` (miembro privado) y en `begin()` con el
   patrón de dirección desde el catálogo:
   ```cpp
   uint8_t addr = registry_ ? ... : pins_.i2cAddrX;
   miSensor_.begin(addr, &Wire);
   ```
4. En `update()`, gatear con el catálogo:
   ```cpp
   if (on("mi_sensor", cfg_.sensorMi)) { ... setValue(...); }
   ```
5. Registrarlo en `SensorRegistry::buildFromConfig()`.
6. Documentar en [Sensores](Sensores.md) + [Compatibilidad](Compatibilidad.md).

## 4. Agregar un expansor

1. Crear el driver en `include/hardware/` + `src/hardware/`.
2. Añadir (si hace falta) las constantes en `PinConfig` + serialización JSON.
3. Agregar el `HardwareKind` correspondiente en `PlatformTypes.hpp`.
4. En `main.cpp`, sumar un pool y una rama en el bucle del catálogo:
   ```cpp
   if (nd.kind == HardwareKind::MI_EXPANSOR && poolCount < 4) {
     miPool[poolCount++].begin(&app.spi, nd.address);
   }
   ```
5. Si es salida, mapear su rango de canales en `ActuatorManager::writeChannel()`.
6. Documentar en [Referencia de código](Referencia-de-codigo.md).

## 5. Agregar un endpoint REST

1. Declarar el handler en `include/api/RestApi.hpp`.
2. Registrar la ruta en `RestApi::begin()`:
   ```cpp
   server_.on("/api/v1/mi-recurso", HTTP_GET, [this](){ handleMiRecurso(); });
   ```
3. Implementar el handler (protegido con `requireAuth()` si escribe):
   ```cpp
   void RestApi::handleMiRecurso() {
     if (!requireAuth()) return;
     server_.send(200, "application/json", "...");
   }
   ```
4. Documentar en [API REST](API-REST.md).

## 6. Agregar un campo de configuración

1. Sumar el campo en `SystemConfig` (`core/Types.hpp`) con su default.
2. Serializar/deserializar en `ConfigManager::toJson` / `fromJson`.
3. Si rompe compatibilidad, subir `GH_CONFIG_SCHEMA_VERSION` y agregar la
   migración en `ConfigManager::migrate()`.
4. Documentar en [Referencia de configuración](Referencia-configuracion.md).

## 7. Agregar una regla de automatización

Las reglas usan `AutomationRule` (variable, operador, umbral, acción) y no
requieren código: se cargan por `POST /api/v1/automation`.
Para una **nueva variable** sí hace falta:
1. Sumar el valor en `RuleVariable`.
2. Leerla en `RuleEngine::readVariable()`.

## 8. Flujo de release

```text
1. Bump GH_FW_VERSION en include/core/Version.hpp
2. Actualizar CHANGELOG.md
3. pio run  (verificar SUCCESS)
4. SHA-256 del firmware.bin → firmware_manifest.json
5. git commit + git tag -a vX.Y.Z + push --tags
6. gh release create vX.Y.Z --notes "..." firmware.bin
7. Actualizar wiki + repo Docs
```

Ver [Reglas de trabajo](../Reglas-de-trabajo.md) y
[Estándar de documentación](../Estandar-de-documentacion.md).

## 9. Checklist antes de un PR/release

- [ ] `pio run` en SUCCESS.
- [ ] Versión + CHANGELOG + manifest (SHA-256).
- [ ] Documentación actualizada (esta wiki + Docs).
- [ ] Sin secretos ni pines hardcodeados.
- [ ] Comentarios del *por qué* en código no obvio.
