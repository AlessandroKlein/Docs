---
tags:
  - sema
  - soporte
---

# Solución de problemas

> **Tipo:** Soporte | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Cada caso sigue el mismo formato: **síntoma → causa probable → cómo confirmarlo →
solución**. Todos los comandos y endpoints citados existen en el firmware v1.103.0.

## 1. Protocolo de diagnóstico (empezar acá)

1. **Monitor serial** a 115200 baudios: es el único canal que muestra el arranque
   completo y los errores previos a la red.

   ```bash
   pio device monitor -b 115200
   ```

2. **Estado general**: `GET /api/v1/health` (`status`, sensores online).
3. **Detalle del sistema**: `GET /api/v1/diagnostics` (heap, `reset_reason`, histórico,
   tareas, módulos, dispositivos I²C).
4. **Identidad y versiones**: `GET /api/v1/system` (firmware, `hw`, `config_schema`,
   `protocol`).
5. **Red**: `GET /api/v1/network` (modo, IP, RSSI).

Si tenés que pedir ayuda, adjuntá la salida de `/api/v1/diagnostics` + las primeras
30 líneas del monitor serial.

---

## 2. Red y conectividad

### 2.1 No conecta a WiFi

| | |
|---|---|
| **Causa probable** | SSID/clave incorrectos, red de 5 GHz (el ESP32 solo usa 2,4 GHz), DHCP sin respuesta o `network.mode` en `AP` |
| **Confirmar** | `GET /api/v1/network` (modo y estado) y `GET /api/v1/wifi/scan` para ver si el SSID aparece |
| **Solución** | Corregir `network.ssid`, `network.password` y `network.mode` con `PUT /api/v1/config`; verificar que la red sea 2,4 GHz; revisar el router |

### 2.2 No aparece `sema-001.local`

| | |
|---|---|
| **Causa probable** | `network.mdns` en `false`, red sin multicast/DNS-SD, o el cliente no soporta mDNS |
| **Confirmar** | `GET /api/v1/network` → `ip` |
| **Solución** | Usar la IP directa; activar `network.mdns`; en Windows instalar el servicio Bonjour si hace falta |

### 2.3 Ethernet sin enlace

| | |
|---|---|
| **Causa probable** | `SEMA_USE_ETHERNET` deshabilitado en el entorno, pines del PHY/SPI mal configurados, o cable/switch |
| **Confirmar** | `GET /api/v1/network` (modo y `connected`); LEDs del conector; monitor serial al arranque |
| **Solución** | Ver [Conectividad y red](Conectividad-y-red.md): LAN8720A por RMII (ESP32-WROOM) o W5500 por SPI (ESP32-S3); revisar `build_flags` y el mapeo de pines |

### 2.4 Perdí la IP de la estación

| | |
|---|---|
| **Causa probable** | DHCP asignó otra IP |
| **Solución** | Buscar en la tabla del router, por nombre `sema-001.local`, o configurar IP estática (`network.ip`, `gateway`, `subnet`, `dns`) y reiniciar |

---

## 3. Configuración

### 3.1 `PUT /api/v1/config` responde `400`

| | |
|---|---|
| **Causa probable** | JSON mal formado o que no pasa la validación (`ConfigManager::validate`) |
| **Confirmar** | El cuerpo de la respuesta indica el error; comparar con [Referencia de configuración](Referencia-configuracion.md) |
| **Solución** | La operación es **transaccional**: si falla, la config anterior queda intacta. Corregir el JSON y reintentar |

### 3.2 La estación arrancó con configuración por defecto

| | |
|---|---|
| **Causa probable** | La config guardada en NVS no se pudo deserializar o no pasó la validación → `ConfigManager` carga los defaults de schema 1 |
| **Confirmar** | Valores en `GET /api/v1/config` distintos de los configurados; líneas de error en el monitor serial |
| **Solución** | Restaurar desde `GET /api/v1/backup` guardado previamente (`POST /api/v1/backup`) |

### 3.3 Perdí la contraseña / no puedo entrar

| | |
|---|---|
| **Causa probable** | `security.api_key`, `security.password` o `security.extra_keys` desconocidos |
| **Solución** | Conservar el backup; si no hay acceso, recuperar por USB (borrado de NVS con `pio run -t erase` → **se pierde toda la configuración**) y volver a configurar. Ver [Seguridad](Seguridad.md) |

---

## 4. Autenticación

| Código | Causa | Solución |
|:------:|-------|----------|
| `401` | Falta `X-API-Key` o la clave es incorrecta | Enviar `X-API-Key: <security.api_key>` (también acepta `server_key`); si ambas están vacías, el acceso está abierto (primera puesta en marcha) |
| `401` en el dashboard | Sin cookie de sesión | `POST /login` con `username`/`password`; la cookie `sema_auth` dura 1 h deslizante |
| `429` | Rate limit del login (5 intentos / 60 s) | Esperar 60 s; corregir la credencial antes de reintentar |
| Sesión que expira | Inactividad > 1 h | Volver a iniciar sesión |
| `403` / escritura rechazada | Endpoints de escritura exigen `X-API-Key`, no sirve la cookie | Usar el header `X-API-Key` |

---

## 5. Sensores

### 5.1 No aparece ningún dispositivo I²C

| | |
|---|---|
| **Causa probable** | SDA/SCL invertidos, sin pull-ups, alimentación ausente o dirección distinta |
| **Confirmar** | `GET /api/v1/diagnostics` → `i2c_devices[]` |
| **Solución** | Pull-ups de 4,7 kΩ a 3,3 V en SDA y SCL; verificar 3,3 V; comprobar direcciones en [Sensores](Sensores.md) |

### 5.2 El DS18B20 no aparece

| | |
|---|---|
| **Causa probable** | Falta la resistencia de pull-up de 4,7 kΩ, o el bus está en el GPIO equivocado |
| **Solución** | Pull-up DATA→3,3 V, GPIO configurado en `sensors[].pin`; los DS18B20 se listan con su ROM (`28-…`) |

### 5.3 Mediciones en error o `SENSOR_DISCONNECTED`

| | |
|---|---|
| **Causa probable** | Sensor desconectado, dirección incorrecta o bus saturado |
| **Confirmar** | `GET /api/v1/health` → comparar `sensors.total` con `sensors.online`; `GET /api/v1/sensors` para ver `quality` por canal |
| **Solución** | Revisar cableado y alimentación; reintentar reconociendo el bus I²C |

### 5.4 La veleta marca direcciones erráticas

| | |
|---|---|
| **Causa probable** | `wind_direction_pin` o `wind_rpull` mal configurados, tabla de resistencias distinta a la del fabricante, o falta de calibración del norte |
| **Solución** | Ajustar `system.wind_resistors[]` (8 valores N→NO), `system.wind_rpull` y `system.wind_north_offset`; usar `POST /api/v1/wind/north` y `POST /api/v1/wind/resistors` desde la página `/config/wind` |

### 5.5 El pluviómetro no cuenta

| | |
|---|---|
| **Causa probable** | `energy.rain_pin` en 0 (deshabilitado) o PCNT mal configurado |
| **Solución** | Configurar `energy.rain_pin` (y `sensors[]` con modelo `PCNT` para publicar la magnitud) |

---

## 6. Buses industriales

| Síntoma | Causa probable | Confirmar | Solución |
|---------|----------------|-----------|----------|
| Modbus sin datos | Baudrate/paridad/slave ID incorrectos, A/B invertidos, sin terminación | `GET /api/v1/modbus` | Ajustar `modbus.*`; 120 Ω en los extremos del RS485; alimentar el transceptor aislado |
| CAN sin tráfico | Baudrate distinto, transceptor sin alimentación, sin terminación | `GET /api/v1/can` | Igualar baudrate, 120 Ω en extremos, revisar TX/RX |
| LoRa sin enlace | Frecuencia/SF/BW distintos entre nodos, antena ausente | `GET /api/v1/lora` | Igualar parámetros; **nunca** operar sin antena |
| Zigbee no responde | ZNP sin firmware, UART cruzada o puerto equivocado | `GET /api/v1/zigbee` | Verificar RX/TX, alimentación y modo del co-procesador (o del MAX14830 si se usa) |

---

## 7. Almacenamiento

| Síntoma | Causa probable | Confirmar | Solución |
|---------|----------------|-----------|----------|
| Histórico vacío | `storage.backend` sin montar, microSD ausente o sin formatear | `GET /api/v1/diagnostics` → `history.entries` | Revisar `storage.backend`, `sd_enabled` y `sd_cs_pin`; formatear la SD en FAT |
| Las gráficas se vacían | Se alcanzó la retención por tiempo | `storage.retention_days` | Aumentar la retención o exportar el CSV antes |
| Eventos perdidos tras reiniciar | El `EventLog` conserva las últimas 100 entradas | `GET /api/v1/events` | Es un buffer acotado por diseño; exportar periódicamente |

---

## 8. Energía y reinicios

| Síntoma | Causa probable | Confirmar | Solución |
|---------|----------------|-----------|----------|
| No despierta del deep sleep | Wake mal configurado (timer/lluvia) | Monitor serial con `W (` y `reset_reason` = 8 | Revisar el perfil energético y `energy.rain_pin`; el GPIO de wake debe tener el nivel correcto |
| Reinicios periódicos | Alimentación insuficiente (picos de WiFi/LoRa) o watchdog | `GET /api/v1/diagnostics` → `reset_reason` (5/6/7) | Fuente con capacidad suficiente; capacitor de desacople; revisar el watchdog |
| Batería marcada baja siempre | `scale`/`offset` del divisor mal calibrados | `GET /api/v1/sensors` (`BATT`) | Calibrar con multímetro: `scale = V_max · (R1+R2)/R2 / 4095` |
| `reset_reason = 4` (panic) | Excepción de firmware o sensor mal inicializado | Monitor serial (backtrace) | Reportar el caso con el backtrace completo |

---

## 9. OTA y actualización

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| `401` al subir | Falta `X-API-Key` | Agregar el header |
| Subida cortada / timeout | Red inestable o binario muy grande | Reintentar por USB; verificar la partición de app del entorno |
| El binario no arranca y vuelve a la versión anterior | Verificación SHA-256 fallida o partición inválida | Usar el `firmware.bin` correcto para la placa; el rollback A/B es automático |
| No hay actualizaciones detectadas | `GET /api/v1/update/check` sin conectividad | Verificar salida a Internet; el chequeo consulta el manifiesto |

Ver [OTA y actualización](OTA-y-Actualizacion.md).

---

## 10. Dashboard y web

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| Pantalla de login en bucle | Cookie bloqueada por el navegador | Permitir cookies para la IP; la cookie es `HttpOnly; SameSite=Strict` |
| Los datos no se actualizan | WebSocket bloqueado por proxy/VPN | Probar en la red local; el dashboard también refresca por `fetch` |
| El layout se ve desarmado | Gridstack no cargó (assets gzip en PROGMEM, v1.102.0) | Recargar con caché limpia; verificar `GET /gridstack-all.min.js` |
| La página de config no guarda | Falta `X-API-Key` o sesión vencida | Volver a iniciar sesión |

---

## 11. Monitor serial

```bash
pio device monitor -b 115200
```

En el arranque se imprimen: versión, `hw`, schema de configuración, resultado de la
carga de config, montaje de LittleFS/SD, inicialización de buses y sensores, y el
resultado de la conexión de red.

---

## Ver también

- [FAQ](FAQ.md) · [Diagnóstico y salud](Diagnostico-y-salud.md) · [Identidad y estados](Identidad-y-estados.md)
- [Conectividad y red](Conectividad-y-red.md) · [Sensores](Sensores.md) · [Compilación y flasheo](Compilacion-y-flasheo.md)
