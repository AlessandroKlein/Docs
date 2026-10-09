---
tags:
  - sema
  - glosario
---

# Glosario

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Todos los términos que aparecen en la documentación de SEMA, en lenguaje simple. Los
que describen algo **no implementado** en v1.103.0 están marcados con ⚠️ y se indica
qué existe hoy.

## 1. Proyecto y arquitectura

| Término | Significado |
|---------|-------------|
| **SEMA** | Sistema de Estación Meteorológica Autónoma: plataforma modular sobre ESP32 que se configura desde el navegador sin recompilar. |
| **Core** | Núcleo de software que orquesta todo (`SemaCore`): storage, configuración, sensores, red, web, scheduler, módulos y eventos. No depende de módulos opcionales. |
| **Módulo** | Unidad funcional con ciclo de vida propio (`Module` en `include/core/Module.hpp`). ⚠️ La interfaz y el registro existen, pero no hay módulos reales todavía. |
| **Ciclo de vida de módulo** | `AVAILABLE → INSTALLED → CONFIGURED → ENABLED → RUNNING`, con `DISABLED`, `UNINSTALLED` y los errores `INSTALL_ERROR`, `CONFIG_ERROR`, `RUNTIME_ERROR`, `UPDATE_ERROR`. |
| **Capability (capacidad)** | Función que la plataforma declara tener (ADC, PCNT, Wi-Fi, CAN, PSRAM, dual core…). El código pregunta capacidades en lugar de preguntar por modelo de chip. |
| **Capability Manager** | Punto único de consulta de capacidades (`CapabilityManager::has()`). |
| **Resource Manager** | Componente que asigna y valida recursos (GPIO, periféricos, buses) detectando conflictos. ⚠️ Especificado; no existe como clase en v1.103.0. |
| **Runtime Manager** | Componente que administra tareas y afinidad sobre FreeRTOS. ⚠️ Existe la abstracción `runtime/Task.hpp`, no un manager. |
| **HAL** | *Hardware Abstraction Layer*: capa que aísla al resto del código del hardware concreto; se apoya en el perfil de placa/chip. |
| **Board / Chip Profile** | Perfil que declara qué puede hacer una placa o chip concreto (pines, features, capacidades). En el código: `include/hw/HwProfile.hpp` y `include/core/BoardProfile.hpp`. |
| **Resource / Capability Matrix** | Tabla que cruza cada placa/chip con las capacidades que ofrece (GPIO, ADC, PCNT, PWM, UART, I²C, SPI, TWAI, Wi-Fi, Bluetooth, 802.15.4, PSRAM, RTC GPIO, deep sleep, flash). |
| **Offline-first** | Principio de diseño: la estación funciona completa sin Internet ni servidor; la nube es complementaria. |
| **Publishers / publicador** | Módulo que envía cada medición a un servicio externo (webhook HTTP, MQTT). Se ejecuta después del almacenamiento y no puede bloquear la adquisición. |
| **Store & Forward** | Encolar datos localmente y reenviarlos al recuperar la conexión. ⚠️ Especificado; no implementado en v1.103.0. |
| **Servidor Central** | Componente multiestación externo que agrega varias SEMA. ❌ Fuera del alcance de SEMA: solo comparte `security.server_key` y `SEMA_PROTOCOL_VERSION`. |
| **Schema version** | Versión del formato del JSON de configuración. Hoy `SEMA_CONFIG_SCHEMA_VERSION = 1`. |
| **Protocol version** | Versión del protocolo de diálogo con el Servidor Central. Hoy `SEMA_PROTOCOL_VERSION = 1`. |
| **Expert Mode** | Modo que habilita la configuración avanzada (perfiles de tareas, prioridades). ⚠️ Ninguna opción de runtime está expuesta todavía. |
| **Safe Mode** | Modo de recuperación (AP + web básica) cuando la configuración es inválida o el arranque falla. ⚠️ Especificado; no implementado como modo de arranque. |
| **Hot reload** | Re-aplicar configuración **sin reiniciar**: sensores, reglas, calibración, GPIO, publicadores y buses. |
| **ADR** | *Architecture Decision Record*: ficha de una decisión con contexto, decisión y consecuencias. Ver [Decisiones](Decisiones.md). |
| **SemVer** | Versionado `MAJOR.MINOR.PATCH` (`1.103.0`). Los tags y releases de Git usan el prefijo `v` minúscula (`v1.103.0`); la constante del firmware va sin `v`. |
| **DoD** | *Definition of Done*: lista de requisitos que debe cumplir una versión para considerarse completa. |
| **Demo (modo)** | Build `-D SEMA_DEMO=1` que genera valores ficticios y un catálogo simulado para probar la web sin sensores conectados. |

## 2. Electrónica, buses y periféricos

| Término | Significado |
|---------|-------------|
| **GPIO** | Pin digital de propósito general: entrada o salida, configurable por la web (`gpio[]`). |
| **ADC** | *Analog to Digital Converter*: convierte una tensión en un número. El ESP32 tiene ADC de 12 bits (0…4095) y SEMA lo usa para batería, CO y radiación solar. |
| **DAC** | *Digital to Analog Converter*: salida analógica. El ESP32 clásico tiene 2 canales; SEMA declara la capacidad pero no la usa. |
| **PWM / LEDC** | Modulación por ancho de pulso; el periférico LEDC del ESP32 la genera por hardware. ⚠️ SEMA declara la capacidad; no hay salidas PWM configurables todavía. |
| **I²C** | Bus serie de dos hilos (SDA/SCL) con direcciones de 7 bits. En SEMA, SDA/SCL son configurables y por defecto GPIO 21/22. |
| **SPI** | Bus serie de 4 hilos (SCK/MISO/MOSI/CS) para W5500, LoRa, microSD y expansores. En WROOM los pines por defecto son SCK 14, MISO 12, MOSI 15. |
| **UART** | Puerto serie asíncrono (RX/TX). SEMA lo usa para PMS5003 (Serial2, 9600 8N1), Modbus RTU y Zigbee. |
| **1-Wire** | Bus de un solo hilo para DS18B20; requiere resistencia de pull-up de 4,7 kΩ y admite varios sensores en el mismo cable. |
| **PCNT** | *Pulse Counter*: periférico de conteo de pulsos por hardware, usado para pluviómetro y anemómetro. SEMA lo configura con la API legacy `pcnt_*` de ESP-IDF, por flanco ascendente, tope 32767. |
| **RS485** | Bus diferencial industrial, robusto a distancia y ruido; en SEMA transporta Modbus RTU. |
| **Modbus RTU** | Protocolo industrial maestro/esclavo sobre RS485. SEMA es **maestro** de un esclavo (hasta 4 registros por lectura, default 9600 baudios, esclavo 1). |
| **CAN / TWAI** | *Two-Wire Automotive Interface*: controlador CAN 2.0 del ESP32. SEMA permite 125 000, 250 000, 500 000 y 1 000 000 bps (default 500 000). |
| **RMII** | *Reduced Media Independent Interface*: interfaz de 50 MHz entre el MAC Ethernet interno del ESP32 y la PHY externa. Sus pines son fijos y no reasignables. |
| **PHY** | Chip que adapta la señal Ethernet al medio físico. Con ESP32 clásico se usa **LAN8720A** por RMII; en placas sin MAC nativa, **W5500** por SPI. |
| **MCP23017** | Expansor de 16 GPIO por I²C. SEMA lo usa desde `GpioManager` cuando un pin declara `expander_addr`. |
| **MCP23S17** | Expansor de 16 GPIO por SPI (pines A0-A7, B0-B7). ⚠️ En v1.103.0 se configura (CS y modo de cada pin) y se muestra en la web; **no hay driver propio**. |
| **74HC595** | Registro de desplazamiento de salida (8 bits, encadenable por SPI). Requiere un pin LATCH (RCLK) por chip. |
| **74HC165** | Registro de desplazamiento de entrada (8 bits, encadenable por SPI). Requiere un pin LATCH (SH/LD) por chip. |
| **ADS1115** | ADC externo de 16 bits y 4 canales por I²C (dirección 0x48), para señales que el ADC interno no resuelve. |
| **MAX14830** | Expansor SPI a 4 UART (puertos U0-U3). ⚠️ SEMA solo configura su chip-select y el puerto; no hay driver propio. |
| **SC18IS602B** | Expansor SPI a I²C. ⚠️ SEMA solo configura su chip-select; no hay driver propio. |
| **Pull-up** | Resistencia que lleva una línea al nivel alto cuando nadie la maneja (por ejemplo 4,7 kΩ en I²C y 1-Wire). |
| **Divisor resistivo** | Dos resistencias que reducen una tensión para medirla (por ejemplo 11:1 para leer una batería de 12 V con el ADC de 3,3 V). |
| **RTC GPIO** | Pin que sigue funcionando durante el deep sleep y puede despertar al chip. Necesario para el wake por lluvia. |
| **Pin reservado** | Pin ocupado por el hardware (SPI, Ethernet RMII), que no se ofrece para sensores ni salidas. |
| **Pluviómetro de cangilones** | Sensor de lluvia que cierra un contacto por cada vuelco de cangilón; cada pulso equivale a una cantidad fija de mm. |
| **Anemómetro** | Sensor de velocidad del viento; típicamente entrega pulsos proporcionales a la velocidad. |
| **Veleta** | Sensor de dirección del viento; el modelo WH-SP-WD se lee como una resistencia variable que SEMA convierte a ángulo con una tabla de 8 resistencias. |
| **Piranómetro** | Sensor de radiación solar global. En SEMA se implementa con el driver `SOLAR` sobre ADC (W/m²). |
| **Abrigo meteorológico** | Gabinete ventilado y a la sombra donde van temperatura y humedad, para que midan el aire y no el sol. |

## 3. Software, datos y almacenamiento

| Término | Significado |
|---------|-------------|
| **Firmware** | Programa que corre en el ESP32. Versión actual: **1.103.0** (`include/core/Version.hpp`). |
| **Driver (de sensor)** | Clase que sabe hablarle a un modelo concreto y devuelve `Measurement`; implementa `Sensor` (`id()`, `model()`, `interface()`, `begin()`, `measure()`, `healthy()`). |
| **SensorManager** | Registra los sensores, los inicializa, lee todos y aplica calibración por canal. |
| **SensorFactory** | Crea un driver a partir del campo `model` de la configuración. Si el modelo es desconocido devuelve `nullptr`. |
| **Medición canónica** | `Measurement`: representación única de una lectura (estación, sensor, canal, magnitud, valor, unidad, calidad, secuencia, timestamp) compartida por adquisición, storage, API y publishers. |
| **Canal (`channel_id`)** | Nombre de la magnitud que entrega un sensor (`temperature`, `humidity`, `pm25`, `rain`…). Un sensor puede aportar varios canales. |
| **Magnitud (`measurement`)** | Qué se mide, con nombre canónico (`temperature`, `pressure`, `light`, `co2`, `wind_speed`…). |
| **Calidad (quality flag)** | Estado de validez de una medición. Valores: `VALID`, `INVALID`, `STALE`, `TIMEOUT`, `OUT_OF_RANGE`, `CALIBRATION_ERROR`, `COMMUNICATION_ERROR`, `SENSOR_DISCONNECTED` (y `UNKNOWN` para un valor no reconocido al deserializar). |
| **Secuencia (`sequence`)** | Número que crece con cada medición de un sensor; permite detectar huecos y ordenar. |
| **Timestamp** | Marca de tiempo. Las mediciones usan epoch en segundos; los eventos usan `millis()` (milisegundos desde el arranque). |
| **Epoch** | Segundos transcurridos desde el 1/1/1970. Es el formato de `timestamp` en la API. |
| **NTP** | Protocolo de sincronización de hora por red. Si no sincronizó, `nowEpoch()` devuelve el uptime (`millis()/1000`) para no devolver 0. |
| **Zona horaria IANA** | Nombre de zona (`America/Argentina/Buenos_Aires`); `SemaCore` lo traduce a una cadena POSIX para el reloj. |
| **Magnitud derivada / derivada** | Valor calculado a partir de otras mediciones, sin sensor propio. En SEMA: `dew_point`, `heat_index`, `vapor_pressure`, `absolute_humidity`, `vpd`, `wind_chill`, `qnh`, `barometric_altitude`, `aqi`, `rain_rate`, `rain_accumulated`, `wind_direction`. |
| **Punto de rocío (`dew_point`)** | Temperatura a la que el aire satura y empieza a condensar. |
| **Índice de calor (`heat_index`)** | Sensación térmica por calor y humedad (regresión de Rothfusz). |
| **Sensación térmica por viento (`wind_chill`)** | Temperatura percibida por efecto del viento. |
| **VPD** | *Vapor Pressure Deficit*: déficit de presión de vapor (kPa), indicador de demanda evaporativa del aire. |
| **QNH** | Presión atmosférica reducida al nivel del mar, calculada con la altitud configurada. |
| **Altitud barométrica** | Altitud estimada a partir de la presión. |
| **AQI** | *Air Quality Index*: índice de calidad del aire derivado de PM2.5 por tramos de la EPA. |
| **Histéresis** | Técnica de control con dos umbrales (encender por debajo de uno, apagar por encima de otro) para evitar oscilación. ⚠️ SEMA solo evalúa umbrales simples; no hay histéresis. |
| **PID** | Control Proporcional-Integral-Derivativo. ⚠️ No implementado en SEMA (ver [Control PID](Control-PID.md)). |
| **Regla (`rules[]`)** | Condición configurable sobre un canal: `gt`, `lt`, `ge` o `le` contra un umbral. |
| **Alarma** | Evento de tipo `Alarm` que publica el motor de reglas cuando una condición se cumple. Queda en el `EventLog`. |
| **Evento** | Notificación tipada del Event Bus: `sensor`, `rain`, `lightning`, `battery`, `network`, `alarm`, `system`, `wake`, `sleep`. |
| **Severidad** | Nivel del evento: `DEBUG`, `INFO`, `NOTICE`, `WARNING`, `ERROR`, `CRITICAL`. |
| **Event Bus** | Bus interno que desacopla quien detecta un hecho de quien reacciona. Sincrónico: el handler corre en el hilo que publica. |
| **EventLog** | Registro persistente de eventos en LittleFS (`/events.jsonl`), con buffer de 100 en RAM y rotación al superar 200 líneas. |
| **HistoryStore** | Almacén del histórico de mediciones en **microSD** (`/history.jsonl` + `/history.jsonl.agg`). Sin microSD no guarda. |
| **JSONL** | Formato de una línea JSON por registro, usado para histórico y eventos. |
| **Rotación** | Cuando el archivo crece demasiado, se conserva la parte reciente y se descarta la vieja (histórico: la mitad; eventos: las últimas 100 líneas). |
| **Retención** | Cuánto tiempo se conservan los datos. `storage.retention_days`, default 30. |
| **Agregación** | Bajar la resolución de lo viejo: se guardan promedios por hora en el archivo `.agg` en lugar de descartarlos. |
| **NVS** | *Non-Volatile Storage*: memoria clave-valor de ESP-IDF. SEMA guarda ahí la configuración (namespace `sema`), el contador `boots` y el layout del dashboard. |
| **LittleFS** | Sistema de archivos en la partición `spiffs` del flash; SEMA lo usa para los eventos. |
| **microSD** | Tarjeta por SPI (CS configurable, default GPIO 4) donde vive el histórico. Es **opcional**. |
| **SPIFFS** | Sistema de archivos antecesor de LittleFS; la partición conserva ese nombre en `partitions_*.csv` aunque se monte LittleFS. |
| **Gridstack** | Biblioteca web del dashboard que permite acomodar y arrastrar tarjetas; su layout se guarda en NVS y se sirve embebida en flash comprimida (gzip). |
| **mDNS** | Resolución de nombres `<hostname>.local` sin servidor DNS. Por defecto `http://sema-001.local/`. |
| **WebSocket** | Canal bidireccional persistente. SEMA difunde las mediciones en el puerto **81**. |
| **OTA** | *Over-The-Air*: actualización de firmware por red, sin cable. |
| **Particiones A/B** | Dos particiones de aplicación (`app0`, `app1`) más `otadata`, que permiten actualizar sin perder la versión que funciona. |
| **SHA-256** | Huella criptográfica de 32 bytes (64 hex) del binario. Se publica en `firmware_manifest.json` y el OTA puede verificarla con la cabecera `X-SHA256`. |
| **Manifiesto (`firmware_manifest.json`)** | Archivo con la versión, el hardware, el esquema, el protocolo y los binarios con su hash por placa. |
| **Watchdog (TWDT)** | Temporizador que reinicia el chip si el programa deja de "alimentarlo". SEMA lo configura a 10 s. |
| **Heartbeat** | Señal periódica de vida. `core.heartbeat` late cada 5 s y `sensors.read` cada 10 s. |
| **Health Monitor** | Componente que calcula el estado de salud (`HEALTHY`, `DEGRADED`, `ERROR`) a partir de sensores online, heartbeat y tareas vigiladas. |
| **Scheduler** | Planificador cooperativo por `millis()` que ejecuta tareas periódicas dentro del `loop()`. |
| **Task / tarea** | Unidad de ejecución. ⚠️ SEMA define una abstracción (`runtime/Task.hpp`) pero hoy **no** crea tareas FreeRTOS: todo corre en el hilo de `loop()`. |
| **Afinidad (`AUTO`)** | Permitir que una tarea corra en cualquier núcleo (`tskNO_AFFINITY`). Es el default de SEMA. |
| **SMP** | *Symmetric Multi-Processing*: uso de los dos núcleos del ESP32 por parte de FreeRTOS. SEMA lo aprovecha si existe, sin depender de él. |
| **Single-core / multicore** | Variantes de ESP32 con uno o dos núcleos; el mismo código debe funcionar en ambas. |

## 4. Red, seguridad y publicación

| Término | Significado |
|---------|-------------|
| **STA** | Modo cliente Wi-Fi: la estación se conecta a un router. |
| **AP** | Modo punto de acceso: la estación crea su propia red (SSID = hostname, sin contraseña) para la primera configuración. |
| **DHCP** | Asignación automática de IP por el router. Es el modo por defecto (IP estática vacía). |
| **IP estática** | IP, gateway, máscara y DNS fijados en la configuración; se aplican solo si los cuatro están completos. |
| **Hostname** | Nombre de la estación en la red (`sema-001` por defecto); de él sale el nombre mDNS. |
| **RSSI** | Potencia de la señal Wi-Fi recibida, en dBm (más cerca de 0 es mejor). |
| **Backoff exponencial** | Reintentar cada vez más espaciado: SEMA reintenta Wi-Fi de 2 s a 60 s como máximo. |
| **REST** | Estilo de API sobre HTTP con métodos (`GET`, `PUT`, `POST`) y respuestas JSON. |
| **API REST** | Interfaz HTTP de SEMA, versionada en `/api/v1`. `HttpServer::begin()` registra **53 rutas**: 11 páginas/acciones web, 40 endpoints de API y 2 assets gzip. |
| **API Key** | Clave enviada en la cabecera `X-API-Key` para autenticar operaciones. |
| **`server_key`** | Clave pensada para que el Servidor Central consulte o configure la estación. |
| **`extra_keys`** | Claves adicionales con nombre en JSON (`{"nombre":"clave"}`), revocables al borrar la entrada. |
| **RBAC** | *Role-Based Access Control*: permisos por rol (administrador, operador, consulta). ⚠️ Especificado; hoy solo hay "autenticado o no". |
| **Sesión web** | Cookie `sema_auth` con token aleatorio, válida 1 h deslizante; se obtiene en `POST /login`. |
| **Rate limiting** | Límite de intentos: 5 logins fallidos bloquean el acceso 60 s (HTTP 429). |
| **TLS / HTTPS** | Cifrado del tráfico. ⚠️ La web y la API de SEMA son HTTP sin cifrar. |
| **Webhook** | URL HTTP que recibe cada medición como JSON (`publishers.webhook_url`). |
| **MQTT** | Protocolo de mensajería publicación/suscripción; SEMA publica en `publishers.mqtt_topic` (default `sema/measurement`). |
| **Broker** | Servidor MQTT al que se conecta la estación (`publishers.mqtt_host`/`mqtt_port`). |
| **Topic** | Canal de MQTT donde se publica. |
| **Update channel** | Canal de actualización (`stable`, `beta`, `development`) que decide qué pre-releases acepta una instalación. |

## 5. Unidades canónicas

| Unidad | Magnitud | Nota |
|--------|----------|------|
| `degC` | temperatura, punto de rocío, índice de calor, sensación térmica | Se convierte a `degF` si `system.units = "imperial"`. |
| `percent` | humedad relativa | 0–100 %. |
| `hPa` | presión, QNH, presión de vapor | Se convierte a `inHg` en imperial (÷ 33,8639). |
| `lux` | luminosidad (BH1750) | — |
| `W/m2` | radiación solar (driver `SOLAR`) | — |
| `µg/m³` | PM1.0, PM2.5, PM10 (PMS5003) | Unidad tal como la reporta la API. |
| `ppm` | CO | — |
| `mm` | lluvia acumulada | Se convierte a `in` en imperial (÷ 25,4). |
| `mm/h` | tasa de lluvia (`rain_rate`) | Se convierte a `in/h`. |
| `mph` | velocidad y ráfaga de viento en imperial | Métrico: `wind_speed` en m/s. |
| `m/s` | velocidad de viento (métrico) | — |
| `deg` | dirección del viento (0–360°, 0 = norte) | — |
| `kPa` | VPD | — |
| `g/m3` | humedad absoluta | — |
| `index` | AQI (adimensional) | — |
| `epoch` | reloj (segundos desde 1970) | Se publica como pseudo-medición `clock`. |
| `dBm` | RSSI de Wi-Fi | — |
| `V` | tensión de batería/ADC | Según `scale` y `offset` del canal. |

## 6. Siglas

| Sigla | Significado |
|-------|-------------|
| **ADC** | *Analog to Digital Converter* |
| **ADR** | *Architecture Decision Record* |
| **API** | *Application Programming Interface* |
| **AQI** | *Air Quality Index* |
| **CAN** | *Controller Area Network* |
| **DAC** | *Digital to Analog Converter* |
| **DoD** | *Definition of Done* |
| **eCO₂** | CO₂ equivalente estimado (SGP30) |
| **GPIO** | *General Purpose Input/Output* |
| **HAL** | *Hardware Abstraction Layer* |
| **I²C** | *Inter-Integrated Circuit* |
| **JSONL** | *JSON Lines* |
| **mDNS** | *multicast Domain Name System* |
| **MQTT** | *Message Queuing Telemetry Transport* |
| **NTP** | *Network Time Protocol* |
| **NVS** | *Non-Volatile Storage* |
| **OTA** | *Over-The-Air* |
| **PCNT** | *Pulse Counter* |
| **PID** | *Proportional-Integral-Derivative* |
| **PM1 / PM2.5 / PM10** | Material particulado de hasta 1, 2,5 y 10 µm |
| **PWM** | *Pulse Width Modulation* |
| **RBAC** | *Role-Based Access Control* |
| **RMII** | *Reduced Media Independent Interface* |
| **RSSI** | *Received Signal Strength Indicator* |
| **RT** | *Real Time* |
| **SMP** | *Symmetric Multi-Processing* |
| **SPI** | *Serial Peripheral Interface* |
| **STA / AP** | *Station* / *Access Point* |
| **TLS** | *Transport Layer Security* |
| **TVOC** | *Total Volatile Organic Compounds* |
| **TWAI** | *Two-Wire Automotive Interface* (CAN del ESP32) |
| **UART** | *Universal Asynchronous Receiver-Transmitter* |
| **UVI** | *UV Index* |
| **VPD** | *Vapor Pressure Deficit* |

## Ver también

- [Home](Home.md) · [Guía de inicio](Guia-de-inicio.md) · [Arquitectura](Arquitectura.md)
- [Enumeraciones y tipos](Enumeraciones-y-tipos.md) · [Referencia de configuración](Referencia-configuracion.md) ·
  [Solución de problemas](Solucion-de-problemas.md) · [FAQ](FAQ.md)
