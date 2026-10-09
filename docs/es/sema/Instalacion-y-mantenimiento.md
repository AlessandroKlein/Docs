---
tags:
  - sema
  - instalacion
  - mantenimiento
---

# Instalación y mantenimiento

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Puesta en marcha de una estación SEMA **en el campo** y plan de mantenimiento: cómo
alimentarla, cómo protegerla, dónde poner cada sensor, cómo dejarla configurada y qué
revisar con el tiempo.

!!! info "Qué es recomendación y qué impone el firmware"
    El firmware **no valida** la instalación física. Los valores de esta página son
    **buenas prácticas de instrumentación meteorológica**; los datos que sí impone SEMA
    (pines por defecto, direcciones I²C, endpoints, escalas) están citados con su
    archivo de código.

## 1. Antes de empezar

- [ ] Firmware compilado y probado en banco (ver [Guía de inicio](Guia-de-inicio.md)).
- [ ] Catálogo de sensores definido: qué mide cada uno y con qué `id`
      (`EXT`, `INT`, `SOIL`, `RAIN`, `WIND`, `BATT`…).
- [ ] Diagrama de conexiones y lista de materiales ([Materiales](Materiales.md),
      [Hardware y conexiones](Hardware-y-Conexiones.md)).
- [ ] Plan de energía: ¿red eléctrica, panel + batería, o ambas?
- [ ] Punto de montaje elegido (ver §5) y forma de acceso para mantenimiento.

## 2. Alimentación y protecciones

| Elemento | Recomendación | Por qué |
|----------|---------------|---------|
| Tensión de la lógica | **3,3 V**. Los GPIO del ESP32 **no** son tolerantes a 5 V | Un nivel de 5 V en un GPIO daña el chip |
| Fuente | 5 V estabilizados ≥ 1 A (o 3,3 V directos) con ripple bajo | El Wi-Fi tiene picos de corriente que reinician la placa si la fuente es débil |
| Capacitor de entrada | 470–1000 µF electrolítico + 100 nF cerámico junto al módulo | Absorbe los picos de transmisión Wi-Fi |
| Protección de polaridad | Diodo Schottky en serie o puente rectificador | Evita destruir el módulo al invertir la alimentación |
| Protección de sobretensión | TVS (por ejemplo SMBJ5.0A) en la entrada | Transitorios de líneas largas y descargas cercanas |
| Fusible | Fusible rearmable o de vidrio dimensionado a la corriente real | Ante un corto evita incendio del gabinete |
| Convertidor a 3,3 V | Regulador con buen rendimiento (buck) si se alimenta de 12 V | Un LDO lineal disipa mucho calor a 12 V |
| Batería | Medir con ADC tras un **divisor resistivo**; el default de SEMA es 11:1 para 12 V | El ADC del ESP32 admite 0–3,3 V (12 bits, 0…4095) |
| Pines de ADC | Usar **ADC1** (GPIO 32–39). ADC2 no es usable mientras el Wi-Fi está activo | Limitación del ESP32, no de SEMA |
| Pines solo-entrada | GPIO 34–39 no tienen salida ni pull-up interno | No sirven para excitar sensores |
| Panel solar | Controlador de carga con MPPT y diodo de bloqueo | Evita descarga nocturna de la batería |
| Protección de línea larga | Fusible + TVS en cada cable que salga del gabinete | Los cables de campo captan descargas inducidas |

Referencias concretas del firmware:

- Batería por defecto: `AdcSensor` en `BATT`, GPIO **34**, `scale = 3,3 × 11 / 4095 ≈
  0,008864` V/count (`src/core/SemaCore.cpp` §290-291). Con `scale`/`offset` se ajusta
  cualquier divisor.
- Los pines de ADC y los conflictos con Wi-Fi se listan en [Guía de pines](Guia-de-pines.md).

!!! danger "Nunca conectes 5 V a un GPIO"
    Sensores de 5 V (por ejemplo algunos PMS5003 o relés) requieren **adaptación de
    nivel** o alimentación propia con la señal adaptada a 3,3 V.

## 3. Gabinete, grado de protección e IP

- **Gabinete**: ABS o policarbonato con tapa, apto para intemperie (IP65/IP66).
  Evitá metal sin protección: es un pararrayos y una jaula que degrada el Wi-Fi.
- **Prensaestopas** en cada entrada de cable (nunca cables por una ranura abierta).
- **Venteo**: un gabinete totalmente sellado condensa agua adentro por el ciclo
  día/noche. Usá venteo hidrofóbico (Gore) o desecante recambiable.
- **Temperatura interna**: el ESP32 y el regulador calientan; dejá aire alrededor y no
  pegues el módulo a la pared del gabinete.
- **Antena**: si usás un módulo con antena externa, dejá el conector accesible y el
  pigtail sin doblar en ángulo cerrado. No encierres la antena dentro de una caja
  metálica.
- **IP fija o reserva DHCP**: para no perder la estación, reservá la IP en el router o
  configurá IP estática (`network.ip`, `network.gateway`, `network.subnet`,
  `network.dns`). SEMA solo aplica la IP estática si **los cuatro** campos están
  completos (`src/core/network/WiFiManager.cpp` §15-22).
- **Etiqueta**: pegá en el gabinete `station.id`, hostname (`sema-001`), IP y fecha de
  instalación. Ahorra la mitad del tiempo de la próxima visita.

## 4. Puesta a tierra y apantallamiento

| Práctica | Detalle |
|----------|---------|
| Referencia común | Un único punto de masa para todo el sistema (evita lazos de tierra). |
| Malla del cable | Conectar la malla **en un solo extremo** (el del gabinete) a la masa. Conectar los dos extremos forma un lazo y mete ruido. |
| Separación de cables | Los cables de señal no corren en paralelo pegado a los de alimentación; cruzarlos a 90° si hay que cruzarlos. |
| Par trenzado | Usar par trenzado para 1-Wire, RS485 y señales de pulsos (lluvia, viento). |
| Longitud I²C | Mantener el bus I²C **corto** (decenas de centímetros; a 100 kHz, ~1 m es el límite práctico). Para distancias mayores, usar un expansor o bajar la frecuencia. |
| Longitud 1-Wire | Puede llegar a decenas de metros con pull-up de **4,7 kΩ** y topología de bus (no estrella). SEMA usa `SEMA_PIN_ONEWIRE = 4`. |
| RS485 | Hasta ~1200 m con par trenzado y **terminación de 120 Ω** en los extremos del bus. SEMA controla DE/RE con `modbus.de_re` (0 = sin control). |
| Pulsos (lluvia/viento) | Cable apantallado y pull-up; el PCNT del ESP32 cuenta por flanco ascendente. Un cable largo sin filtro capta ruido y cuenta pulsos falsos. |
| Descargas | Si el punto es alto o hay tormentas frecuentes, agregá protector gaseoso/TVS en las líneas de campo. |
| Ferrites | Un ferrite de clip cerca del gabinete en cada cable de sensores reduce el ruido de modo común. |

!!! warning "El ruido se ve como datos, no como errores"
    Un pulso falso entra como lluvia real y una lectura de ADC ruidosa como una
    variación de batería. Revisá las series en el dashboard antes de dar por buena una
    instalación con cables largos.

## 5. Ubicación de los sensores

Como regla general, cada sensor mide **el aire o el suelo que lo rodea**, no el
gabinete. Separá los sensores del gabinete y del suelo según corresponda.

| Sensor / magnitud | Dónde | Altura típica | Cuidados |
|-------------------|-------|---------------|----------|
| Temperatura y humedad (BME280, SHT40, SHT31, AHT20, SCD30) | **Abrigo meteorológico** ventilado y a la sombra, lejos de paredes, techos y fuentes de calor | 1,5–2 m sobre el suelo | El abrigo debe ventilar; sin él la lectura es la del aire calentado por el sol |
| Presión (BME280, BMP280) | Dentro del abrigo; la altitud se compensa en la config | — | Configurá `system.altitude` para QNH y altitud barométrica |
| Anemómetro (velocidad de viento, PCNT) | Mástil despejado, sin obstáculos alrededor | 10 m es el estándar meteorológico; si no es posible, lo más alto y libre que permita el sitio | Registrá la altura real: el viento a 2 m no es comparable con el de 10 m |
| Veleta (dirección, ADC) | En el mismo mástil que el anemómetro, por encima de obstáculos | Igual que el anemómetro | Orientar al norte y ajustar con `system.wind_north_offset` (o el endpoint de auto-calibración) |
| Pluviómetro de cangilones (PCNT) | Poste firme, **nivelado**, con el orificio libre de obstáculos verticales | ~1 m sobre el suelo, sobre base rígida | Nivelar con burbuja; un pluviómetro inclinado subestima la lluvia. Evitar salpicadura del suelo |
| Piranómetro (driver `SOLAR`, ADC) | Superficie **horizontal**, sin sombras de mástiles ni del gabinete | Lo más alto posible | Limpiar el domo: polvo y excrementos de aves cambian la lectura |
| PM (PMS5003) | A la sombra, con entrada de aire libre y protegida de lluvia | 1,5–3 m | Lejos de escape de motores, humo de cocina y polvo del camino |
| CO₂ (SCD30) | En el ambiente a medir, ventilado | Según el caso | El SCD30 es NDIR: no lo pongas dentro de una caja cerrada |
| UV (VEML6075) | Horizontal y sin sombras | — | Igual que el piranómetro: horizontalidad y limpieza |
| Rayos (AS3935) | Alejado de fuentes de ruido eléctrico (motores, switching, cables de red) | — | Tiene su propia rutina de calibración de antena, que SEMA no ejecuta |
| Suelo / DS18B20 | Enterrado a la profundidad de interés (raíces, 20/40/60 cm) | — | Cable resistente a humedad; sellar la punta y el empalme |
| Batería / ADC | Cerca de la batería, con cable corto | — | El divisor consume; usar resistencias altas o desactivar la medición si no se usa |

!!! warning "La veleta no se auto-orienta"
    SEMA convierte la resistencia de la veleta WH-SP-WD a un ángulo **relativo** y le
    aplica `system.wind_north_offset` (`DerivedCalculator::windDirection()`). Si no
    orientás el pluviómetro/veleta al norte y ajustás el offset, la dirección será
    incorrecta.

## 6. Primera configuración

1. **Alimentar y ver el arranque** por serie (115200). Confirmá el banner:
   `SEMA v1.103.0 (hw rev0, schema 1, protocol 1)` y la lista de dispositivos I²C
   detectados.
2. **Entrar a la web**. Si la estación no tiene SSID configurado, arranca en modo AP:
   el SSID es el hostname (`sema-001`). Averiguá la IP con
   `GET http://<ip>/api/v1/network`.
3. **Red**: cargar SSID, contraseña, hostname y mDNS en `/config/network`. Si usás IP
   estática, completá **IP, gateway, máscara y DNS**.
4. **Seguridad**: en `/config/security` definí `username` y `password` del login web
   (default de usuario: `admin`), y las claves `api_key` y `server_key` si vas a
   integrar la estación con otros sistemas. Sin claves, **todo queda abierto**.
5. **Sistema**: zona horaria IANA (default `America/Argentina/Buenos_Aires`), servidor
   NTP, unidades (`metric`/`imperial`), altitud real del sitio y parámetros de la
   veleta (`wind_direction_pin`, `wind_rpull`, `wind_resistors`).
6. **Sensores**: en `/config/sensors` declarar cada sensor con su `id`, `model` y pines,
   y **habilitarlo** (`enabled: true`). Comparar contra la detección I²C:
   `GET /api/v1/diagnostics` → `i2c_devices`.
7. **Calibrar la veleta**: orientar al norte y usar `POST /api/v1/wind/north`
   (toma el ángulo actual como norte) y, si cambiás la resistencia de pull-up o los
   valores de la tabla, `POST /api/v1/wind/resistors`.
8. **Calibración por canal**: en `calibrations[]`, ajustar `gain`, `offset` y rango de
   los canales que lo necesiten (ver [Calibración](Calibracion.md)).
9. **Almacenamiento e histórico**: si hay microSD, activar `storage.sd_enabled` y
   `storage.sd_cs` (default GPIO 4), y fijar `storage.retention_days`. **Sin microSD no
   hay histórico**.
10. **Publicadores**: webhook (`publishers.webhook_url`) y/o MQTT
    (`mqtt_host`, `mqtt_port`, `mqtt_topic`, `mqtt_user`, `mqtt_pass`).
11. **Reglas**: definir en `rules[]` los umbrales que deban generar alarma
    (operadores `gt`, `lt`, `ge`, `le`).
12. **Respaldo**: descargar `GET /api/v1/backup` y guardar el JSON con la fecha. Es la
    referencia para restaurar la estación en minutos.

## 7. Verificación con endpoints de diagnóstico

Con la estación en la red, esta rutina confirma que quedó bien instalada:

| Paso | Comando | Qué tiene que mostrar |
|------|---------|-----------------------|
| 1 | `GET /api/v1/status` | `firmware` correcto (`1.103.0`) y `uptime_s` creciendo |
| 2 | `GET /api/v1/health` | `status` en `HEALTHY` (o `DEGRADED` con motivo claro) y heap libre razonable |
| 3 | `GET /api/v1/sensors` | Catálogo completo con `healthy: true` y valores con sentido físico |
| 4 | `GET /api/v1/diagnostics` | `i2c_devices` coincide con lo conectado; `history.entries` crece con microSD |
| 5 | `GET /api/v1/network` | `connected: true`, IP esperada y `rssi` aceptable |
| 6 | `GET /api/v1/system` | Board correcta, `pins_from_file`/`demo` en `false`, pines reservados coherentes |
| 7 | `GET /api/v1/energy` | `profile` y `wake_reason` informados |
| 8 | `GET /api/v1/events` | Evento `system`/`boot` y las alarmas esperadas |
| 9 | `GET /api/v1/update/check` | `current: "1.103.0"` y, si hay release nueva, `update: true` |
| 10 | Dashboard `/` | Mediciones en vivo actualizándose cada 5 s |
| 11 | `ws://<host>:81` | Llegan mensajes `{"type":"measurements","data":[…]}` cada 10 s |
| 12 | `GET /api/v1/history?limit=50` | Con microSD activa, devuelve registros |

!!! tip "Guardá la respuesta de `/api/v1/system`"
    Es la radiografía de la instalación (board, flash, pines reservados) y evita
    discusiones cuando algo no funciona meses después.

## 8. Mantenimiento periódico

| Tarea | Frecuencia sugerida | Cómo |
|-------|--------------------|------|
| Limpieza del pluviómetro | Mensual y después de cada evento fuerte | Retirar hojas/insectos del embudo y de los cangilones; verificar que bascula libre |
| Limpieza del piranómetro y del UV | Mensual | Paño suave y agua; **no** usar solventes |
| Revisión del abrigo meteorológico | Trimestral | Limpiar el filtro/rejillas, verificar que el sol no entra directo al sensor |
| Revisión de la veleta y el anemómetro | Trimestral | Que giren libres; reapretar el mástil; verificar el norte |
| Estado de la batería | Mensual (según `BATT`) | Si la tensión de reposo baja del umbral del banco, reemplazar; revisar bornes sulfatados |
| Revisión de conectores y prensaestopas | Trimestral | Que no entre agua, que los borneras estén firmes (con la estación apagada) |
| Desecante / venteo | Cada visita | Recambiar desecante; verificar que el venteo no esté tapado |
| Backup de configuración | Después de cada cambio | `GET /api/v1/backup` → archivo con fecha |
| Export del histórico | Mensual o por campaña | `GET /api/v1/history?limit=3000&format=csv` |
| Actualización OTA | Cuando haya release, con criterio | Ver §9; nunca en medio de una tormenta |
| Verificación de versión y manifiesto | Cada visita | `GET /api/v1/update/check` y comparar con el CHANGELOG |
| Revisión de eventos y alarmas | Cada visita | `GET /api/v1/alarms` y el dashboard: patrones de falsas alarmas o caídas |
| Limpieza del gabinete | Semestral | Revisar telarañas, nidos y corrosión; verificar drenajes |

## 9. Actualización OTA en campo

```bash
# 1) Ver qué versión publica el repositorio
curl http://<ip>/api/v1/update/check

# 2) Descargar el binario del release y su hash, y enviarlo
curl -X POST http://<ip>/api/v1/ota \
  -H "X-API-Key: <api_key>" \
  -H "X-SHA256: <sha256 de 64 hex del binario>" \
  -F "firmware=@sema_1.103.0_esp32-wroom-4mb.bin"

# 3) La estación responde {"ok":true} y reinicia
```

- El binario debe ser el de la **board correcta** (`firmware_manifest.json` lista
  `board`, `chip` y `flash_mb`).
- Con `X-SHA256` la estación verifica la partición escrita y **no reinicia** si el hash
  no coincide (`400 {"error":"sha256 mismatch"}`).
- Si el OTA falla, la partición anterior sigue arrancando (ver
  [OTA y actualización](OTA-y-Actualizacion.md)).

!!! danger "No actualices sin respaldo"
    Antes de cada OTA: `GET /api/v1/backup` y, si hay microSD, copiá
    `/history.jsonl` y `/history.jsonl.agg`. Una actualización no borra la
    configuración (vive en NVS), pero el respaldo cuesta un minuto y ahorra una visita.

## 10. Checklists

### 10.1 Puesta en marcha

- [ ] Alimentación estable, protegida y con capacitor de entrada.
- [ ] Gabinete cerrado, con prensaestopas y venteo/desecante.
- [ ] Masa única y mallas conectadas en un solo extremo.
- [ ] Sensores en su ubicación correcta (§5) y alejados del gabinete.
- [ ] Pluviómetro nivelado; veleta orientada al norte.
- [ ] Firmware v1.103.0 flasheado y arranque verificado por serie.
- [ ] Red configurada (STA o AP), hostname y mDNS; IP estable.
- [ ] Login web y claves definidos (no dejar la estación abierta).
- [ ] Catálogo `sensors[]` completo y **habilitado**.
- [ ] Zona horaria, altitud y unidades correctas.
- [ ] Calibración de canales y offset de veleta aplicados.
- [ ] microSD activada si se quiere histórico; retención definida.
- [ ] Publicadores y reglas configurados.
- [ ] Respaldo descargado y archivado.
- [ ] Los 12 pasos de §7 verificados.
- [ ] Etiqueta con `station.id`, hostname, IP y fecha.

### 10.2 Visita de mantenimiento

- [ ] Inspección visual: agua, corrosión, nidos, cables pelados.
- [ ] Limpieza de pluviómetro, piranómetro y abrigo.
- [ ] Verificación de giro de anemómetro y veleta.
- [ ] Tensión de batería y estado de bornes.
- [ ] `GET /api/v1/health` y `GET /api/v1/diagnostics` sin novedades.
- [ ] `GET /api/v1/alarms` revisado.
- [ ] Export del histórico y respaldo de configuración.
- [ ] Versión comparada con `update/check`.
- [ ] Desecante recambiado.
- [ ] Registro de la visita (fecha, tareas, hallazgos).

### 10.3 Actualización de firmware

- [ ] Respaldo de configuración e histórico.
- [ ] Release identificada y binario de la board correcta.
- [ ] SHA-256 del binario a mano.
- [ ] Ventana de baja actividad y clima tranquilo.
- [ ] OTA enviado y `{"ok":true}` recibido.
- [ ] Post-actualización: `status`, `health`, `sensors` y `diagnostics` verificados.
- [ ] Página del [CHANGELOG](CHANGELOG.md) revisada para saber qué cambió.

## Ver también

- [Guía de inicio](Guia-de-inicio.md) · [Compilación y flasheo](Compilacion-y-flasheo.md) ·
  [Hardware y conexiones](Hardware-y-Conexiones.md) · [Guía de pines](Guia-de-pines.md)
- [Materiales](Materiales.md) · [Energía y consumo](Energia-y-consumo.md) ·
  [OTA y actualización](OTA-y-Actualizacion.md) · [Backup y restauración](Backup-y-restauracion.md)
- [Configuración](Configuracion.md) · [Calibración](Calibracion.md) ·
  [Solución de problemas](Solucion-de-problemas.md)
