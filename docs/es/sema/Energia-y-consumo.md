---
tags:
  - sema
  - energia
  - consumo
---

# Energía y consumo

> **Tipo:** Referencia
> **Estado:** En desarrollo
> **Fecha:** 2026-10-08
> **Firmware:** v1.103.0

Perfiles energéticos, medición de batería, deep sleep, wake reasons y lo que falta para una
estación autónoma. Fuentes: `include/core/PowerManager.hpp`, `src/core/PowerManager.cpp`,
`include/core/ConfigManager.hpp`, `src/core/ConfigManager.cpp`, `src/core/SemaCore.cpp`,
`src/core/sensors/AdcSensor.cpp`, `src/core/sensors/PcntSensor.cpp`,
`include/core/BoardProfile.hpp`, `src/core/web/HttpServer.cpp` (líneas 2017–2024),
`README.md` §36–37 y §136–137.

---

## 1. Perfiles energéticos

```cpp
// include/core/PowerManager.hpp
enum class EnergyProfile : uint8_t {
  Performance, Normal, LowPower, UltraLowPower
};
```

| Enum | Nombre en `/api/v1/energy` | Qué hace el firmware con él |
|------|---------------------------|-----------------------------|
| `Performance` | `performance` | Nada (solo se reporta) |
| `Normal` | `normal` | Nada — es el valor por default |
| `LowPower` | `low_power` | Nada |
| `UltraLowPower` | `ultra_low_power` | Nada |

```cpp
EnergyProfile profile() const { return profile_; }
void setProfile(EnergyProfile p) { profile_ = p; }
```

`PowerManager` es un singleton (`PowerManager::instance()`); `profile_` arranca en
`EnergyProfile::Normal` y **`setProfile()` no se llama desde ningún punto del firmware**.
Tampoco existe una clave de configuración para el perfil: `Config` no tiene un campo
`energy.profile`. En la práctica `GET /api/v1/energy` siempre responde `"profile": "normal"`.

!!! warning "El perfil no cambia ningún comportamiento"
    `README.md` §137 define qué debería implicar cada perfil (frecuencia de medición,
    Wi-Fi permanente o por intervalos, deep sleep) y `D-0054` los asocia a fuentes de
    wake-up, pero en el código **el perfil no se lee en ningún lazo de control**: el ciclo
    de sensores es fijo (10 s, `Scheduler`), la conectividad no se apaga y nadie entra en
    deep sleep (§4). Es hoy un valor informativo.

---

## 2. Medición de batería por ADC

El catálogo fijo registra la batería como un sensor `ADC` genérico
(`src/core/SemaCore.cpp`):

```cpp
// Batería (ADC interno): divisor 11:1 para 12 V (README §37).
static AdcSensor battery("BATT", SEMA_PIN_BATTERY_ADC, "voltage", "V",
                         3.3f * 11.0f / 4095.0f, 0.0f);
```

| Parámetro | Valor | Fuente |
|-----------|-------|--------|
| Id lógico | `BATT` | `SemaCore.cpp` |
| Pin | **GPIO 34** (`SEMA_PIN_BATTERY_ADC`, ADC1_CH6, solo entrada) | `include/core/BoardProfile.hpp` |
| Resolución | 12 bits → 0…4095 (`analogReadResolution(12)`) | `AdcSensor::begin()` |
| Canal y magnitud | `voltage` (ambos) | `AdcSensor::measure()` |
| Unidad | `V` | ídem |
| Escala | `3,3 × 11 / 4095` = **0,0088645 V/count** | ídem |
| Offset | `0,0` | ídem |
| Fondo de escala | 36,3 V (11 × 3,3 V) | derivado del divisor 11:1 |

Fórmula aplicada en el driver: `value = raw · scale + offset`. Ejemplos
*(cálculo propio sobre esos valores)*:

| `raw` | Tensión de batería |
|-------|--------------------|
| 1000 | 8,86 V |
| 1240 | 10,99 V |
| 1350 | 11,97 V |
| 1500 | 13,30 V |
| 1860 | 16,49 V |

Con `SEMA_FIXED_HARDWARE=0` (default), la batería se configura como cualquier otro sensor
analógico (`sensors[]` con `"model": "ADC"`, `pin`, `channel: "voltage"`, `unit: "V"` y el
`scale` del divisor propio). El ejemplo de `scale` que trae el repo es `0.00887`, el mismo
divisor 11:1 redondeado.

La tensión se consulta por:

```bash
curl -s http://sema.local/api/v1/sensors | jq '.measurements[] | select(.sensor_id=="BATT")'
# { "sensor_id": "BATT", "channel_id": "voltage", "measurement": "voltage",
#   "value": 13.3, "unit": "V", "quality": "VALID", "sequence": 412 }
```

y queda registrada en el histórico (`sensor: "BATT"`, `channel: "voltage"`).

!!! note "No hay porcentaje, ni corriente, ni energía"
    El firmware mide **tensión** y nada más: no calcula estado de carga (%), no integra
    energía (Wh), no mide corriente y no publica un evento `EventType::Battery` (el tipo
    existe en `include/core/EventBus.hpp` pero ningún módulo lo emite). Tampoco hay una
    alarma de batería baja de fábrica: hay que definirla como regla (ver
    [Alarmas y reglas §7](Alarmas-y-reglas.md)).

---

## 3. Deep sleep

```cpp
// src/core/PowerManager.cpp — implementación completa de la API de energía
void PowerManager::sleep(uint64_t seconds) {
  esp_sleep_enable_timer_wakeup(seconds * 1000000ULL);   // wake por timer RTC (D-0021)
  esp_deep_sleep_start();
}

void PowerManager::enableRainWakeup(uint8_t pin) {
  // Pluviómetro de cangilones; requiere pin RTC-capable; nivel HIGH en el pulso.
  esp_sleep_enable_ext0_wakeup(static_cast<gpio_num_t>(pin), 1);
}

uint32_t PowerManager::wakeReason() const {
  return static_cast<uint32_t>(esp_sleep_get_wakeup_cause());
}
```

| Función | Estado real |
|---------|-------------|
| `sleep(seconds)` | Implementada (timer RTC + `esp_deep_sleep_start()`), **sin un solo llamador** en `src/` |
| `enableRainWakeup(pin)` | Implementada (`ext0`, nivel alto); se invoca **una vez** en `setup()` |
| `wakeReason()` | Implementada y usada (serial + `/api/v1/energy`) |

`SemaCore::setup()` arma el wake por lluvia solo si hay pin configurado:

```cpp
Serial.printf("Wake reason: %u\n", PowerManager::instance().wakeReason());
if (config_.get().energy.rainPin != 0) {
  PowerManager::instance().enableRainWakeup(config_.get().energy.rainPin);
}
```

`energy.rain_pin` default es `0` (deshabilitado), así que de fábrica **no** se arma nada.

!!! danger "Hoy la estación nunca duerme"
    El ciclo dormir→despertar existe como API, pero ninguna tarea, módulo ni perfil llama a
    `PowerManager::sleep()`. Consecuencias prácticas: `wake_reason` es siempre 0 (arranque
    normal), el wake por `ext0` queda armado pero inerte (sin `esp_deep_sleep_start()` no
    hay reposo) y el perfil `ultra_low_power` no reduce el consumo. Es el pendiente
    estructural de energía.

**Requisitos del wake por GPIO (`ext0`)**: el pin debe ser RTC-capable y el pulso llega en
nivel **alto** (`1`). El comentario del código ejemplifica GPIO4/5; los GPIO 34–39
(solo entrada, RTC-capable) también califican. El pluviómetro, además, necesita su propio
sensor `PCNT` con `channel: "rain"` para contar los pulsos: el wake y el conteo son cosas
separadas.

---

## 4. Wake reasons

`wakeReason()` devuelve crudo el valor de `esp_sleep_get_wakeup_cause()` como `uint32_t`.
Valores del `esp_sleep_wakeup_cause_t` de la versión de ESP-IDF que usa este repo
(`framework-arduinoespressif32`, `esp_hw_support/include/esp_sleep.h`):

| Valor | Constante | Significado |
|-------|-----------|-------------|
| 0 | `ESP_SLEEP_WAKEUP_UNDEFINED` | El reset **no** vino de salir de deep sleep (arranque normal) |
| 1 | `ESP_SLEEP_WAKEUP_ALL` | No es una causa: se usa para deshabilitar todas las fuentes |
| 2 | `ESP_SLEEP_WAKEUP_EXT0` | Señal externa por RTC_IO (es el que usa el wake por lluvia) |
| 3 | `ESP_SLEEP_WAKEUP_EXT1` | Señal externa por RTC_CNTL |
| 4 | `ESP_SLEEP_WAKEUP_TIMER` | Timer RTC (es el que usaría `sleep(seconds)`) |
| 5 | `ESP_SLEEP_WAKEUP_TOUCHPAD` | Touchpad |
| 6 | `ESP_SLEEP_WAKEUP_ULP` | Programa ULP |
| 7 | `ESP_SLEEP_WAKEUP_GPIO` | GPIO (solo light sleep en ESP32/S2/S3) |
| 8 | `ESP_SLEEP_WAKEUP_UART` | UART (solo light sleep) |
| 9 | `ESP_SLEEP_WAKEUP_WIFI` | Wi-Fi (solo light sleep) |
| 10 | `ESP_SLEEP_WAKEUP_COCPU` | Interrupción del co-CPU |
| 11 | `ESP_SLEEP_WAKEUP_COCPU_TRAP_TRIG` | Crash del co-CPU |
| 12 | `ESP_SLEEP_WAKEUP_BT` | Bluetooth (solo light sleep) |

SEMA configura solo dos fuentes: **timer** (`sleep()`) y **`ext0`** (`enableRainWakeup()`).
En el arranque el valor se imprime en el serial (`Wake reason: N`) y se expone por HTTP.

---

## 5. `GET /api/v1/energy`

```cpp
void HttpServer::onEnergy() {
  DynamicJsonDocument doc(256);
  doc["profile"] = energyProfileName(core_->power().profile());
  doc["wake_reason"] = core_->power().wakeReason();
  server_.send(200, "application/json", out);
}
```

```json
{ "profile": "normal", "wake_reason": 0 }
```

| Aspecto | Comportamiento |
|---------|----------------|
| Claves | Exactamente dos: `profile` (string) y `wake_reason` (entero) |
| Autenticación | **Ninguna** (`onEnergy()` no llama a `webAuthed()`) |
| Tensión de batería | ❌ No está en este endpoint: se lee de `/api/v1/sensors` o del histórico |
| Estado de carga, corriente, potencia | ❌ No existen |
| Consumo acumulado | ❌ No existe |

---

## 6. Autonomía y consumo: qué se puede y qué no se puede afirmar

**No hay ningún dato de consumo eléctrico en el repositorio**: ni corrientes en mA, ni
capacidad de batería en Ah, ni curvas de consumo por perfil, ni medición de corriente. Todo
lo que se lea en internet sobre autonomía de un ESP32 es un supuesto externo, no un dato
verificado de SEMA.

Lo único verificable en el repo para estimar autonomía es la **tasa de escritura** (y por lo
tanto el tiempo que el equipo pasa despierto si algún día se implementa el ciclo de sleep):
`SemaCore` dispara `sensors.read` cada 10 000 ms y cada ciclo produce 14 registros de
histórico (ver [Almacenamiento §10](Almacenamiento-e-historico.md)).

**Modelo de estimación** *(cálculo propio, no del repo — usalo solo como plantilla)*:

```text
t_autonomía[h] = (C_batería[Ah] · DoD · η) / I_promedio[A]

I_promedio = (t_activo · I_activo + t_sleep · I_sleep) / (t_activo + t_sleep)
```

| Término | De dónde sale |
|---------|---------------|
| `C_batería`, `DoD`, `η` | Datos de la batería y del regulador: **fuera del repo** |
| `I_activo`, `I_sleep` | Medición con instrumental: **fuera del repo** |
| `t_activo`, `t_sleep` | Del diseño del ciclo (hoy: siempre activo, `sleep()` sin llamadores) |

Sin `I_activo`, `I_sleep` y `C_batería` medidos en el hardware real, cualquier número de
autonomía es una invención: **no se documenta ninguno**.

---

## 7. Lo que falta (y por dónde resolverlo)

| Falta | Estado | Vía de solución |
|-------|--------|-----------------|
| Entrar en deep sleep | ❌ `sleep()` sin llamadores | Un módulo o el `Scheduler` que, cumplido el perfil, publique `EventType::Sleep`, cierre publicadores y llame a `power().sleep(segundos)` |
| Perfil aplicado al runtime | ❌ `setProfile()` sin llamadores | Campo `energy.profile` en config + lectura en el scheduler (intervalo de `sensors.read`), Wi-Fi por intervalos y decisión de dormir |
| Clave de configuración del perfil | ❌ No existe en `Config` | Agregar `energy.profile` (string) con default `"normal"` |
| Wake por timer distinto del ciclo | ❌ | `sleep(seconds)` ya lo soporta; falta el intervalo configurable |
| Light sleep | ❌ (`README.md` §136) | `esp_light_sleep_start()` no está usado en ningún punto |
| Alarma de batería baja / evento `Battery` | ❌ | Regla `rules[]` sobre `BATT`/`voltage` + (si se quiere evento propio) publicar `EventType::Battery` desde el lazo de sensores |
| Estado de carga (%) y energía (Wh) | ❌ | Curva de descarga por química en config + integración de `voltage` en el tiempo |
| Medición de corriente | ❌ | Sensor `ADS1115`/`ADC` sobre un shunt de alta/baja; `sensors[]` ya lo soporta sin firmware nuevo |
| Tensión del panel solar | ⚠️ Se puede medir como sensor `ADC` (`channel: "voltage"`), pero no hay driver de panel | Agregar una entrada `sensors[]` con divisor propio; `SolarSensor` es para **radiación** (`solar_radiation` en `W/m2`), no para tensión de panel |
| **MPPT / controlador de carga solar** | ❌ No implementado | No hay driver de MPPT ni de regulador. Dos caminos reales con lo que ya existe: (a) un controlador con salida **Modbus RTU** leído por el `ModbusManager` (`modbus.slave_id`, `register`, `count`, `baud` — hoy **un solo esclavo**, lecturas a registros consecutivos); (b) un controlador con salida RS485/serial propia, documentado como módulo externo (D-0060). En ambos casos, la telemetría entraría como mediciones y desde ahí a histórico, MQTT y reglas |
| Corte por batería baja | ❌ | Acción local fuera del firmware, vía `POST /api/v1/gpio` (ver [Actuadores y salidas](Actuadores-y-Salidas.md)) |

---

## Ver también

- [Almacenamiento e histórico](Almacenamiento-e-historico.md) · [Alarmas y reglas](Alarmas-y-reglas.md)
- [Calibración](Calibracion.md) · [Magnitudes derivadas](Magnitudes-derivadas.md)
- [API REST](API-REST.md) · [Referencia de configuración](Referencia-configuracion.md)
- [Identidad y estados](Identidad-y-estados.md) · [Hardware y conexiones](Hardware-y-Conexiones.md)
- [Mejoras y roadmap](Mejoras-y-roadmap.md) · [Futuro](Futuro.md)
