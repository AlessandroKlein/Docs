---
tags:
  - sema
  - cambios
---

# Registro de cambios por archivo

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Qué archivos cambió cada release y **por qué** (mensaje de commit que lo motivó). Es el
complemento por archivo del [CHANGELOG](CHANGELOG.md): el CHANGELOG resume *qué cambió*
en cada versión, este registro dice *en qué archivos* y con qué commit.

- **v1.34.0 → v1.103.0**: ficha generada desde el historial real de git
  (`git log --name-only`) — una fila por archivo con el/los commits que lo tocaron.
- **v1.6.0 → v1.33.0**: fichas curadas a mano (formato original).
- Las filas `CHANGELOG.md · firmware_manifest.json · include/core/Version.hpp · docs/*`
  agrupan el trámite de release (bump de versión, manifiesto, documentación).
- Para los detalles funcionales de cada versión, usar el [CHANGELOG](CHANGELOG.md);
  para las convenciones, [Versionado](../inicio/Versionado.md).

---

## v1.103.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): MAX14830 con 4 puertos UART |
| `src/core/ConfigManager.cpp` | feat(web): MAX14830 con 4 puertos UART |
| `src/core/web/HttpServer.cpp` | feat(web): MAX14830 con 4 puertos UART |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): MAX14830 con 4 puertos UART |



## v1.102.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `gen_gridstack.py` | chore(web): eliminar data/ + onStaticFile/LittleFS del web |
| `include/core/web/HttpServer.hpp` | chore(web): eliminar data/ + onStaticFile/LittleFS del web |
| `src/core/web/HttpServer.cpp` | chore(web): eliminar data/ + onStaticFile/LittleFS del web |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | chore(web): eliminar data/ + onStaticFile/LittleFS del web |


## v1.101.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): selector bus I2C/UART por expansores SPI |
| `src/core/ConfigManager.cpp` | feat(web): selector bus I2C/UART por expansores SPI |
| `src/core/web/HttpServer.cpp` | feat(web): selector bus I2C/UART por expansores SPI |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): selector bus I2C/UART por expansores SPI |


## v1.100.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `gen_gridstack.py` | feat(web): gridstack embebido en PROGMEM con gzip |
| `src/core/web/GridstackAssets.h` | feat(web): gridstack embebido en PROGMEM con gzip |
| `src/core/web/HttpServer.cpp` | feat(web): gridstack embebido en PROGMEM con gzip |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): gridstack embebido en PROGMEM con gzip |


## v1.99.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): mostrar expansores con CS en Pines de buses |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): mostrar expansores con CS en Pines de buses |


## v1.98.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): expansores SPI MAX14830/SC18IS602B |
| `src/core/ConfigManager.cpp` | feat(web): expansores SPI MAX14830/SC18IS602B |
| `src/core/web/HttpServer.cpp` | feat(web): expansores SPI MAX14830/SC18IS602B |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): expansores SPI MAX14830/SC18IS602B |


## v1.97.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `include/hw/HwProfile.hpp` | fix(hw): reservados visibles + SPI WROOM remapeado |
| `src/core/ConfigManager.cpp` | fix(hw): reservados visibles + SPI WROOM remapeado |
| `src/core/web/HttpServer.cpp` | fix(hw): reservados visibles + SPI WROOM remapeado |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(hw): reservados visibles + SPI WROOM remapeado |


## v1.96.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): I2C con guardado propio + reinicio |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): I2C con guardado propio + reinicio |


## v1.95.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(hw): reservar SPI + RMII condicional |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(hw): reservar SPI + RMII condicional |


## v1.94.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): pines de buses editables + SD CS movido |
| `include/core/web/HttpServer.hpp` | feat(web): pines de buses editables + SD CS movido |
| `src/core/ConfigManager.cpp` | feat(web): pines de buses editables + SD CS movido |
| `src/core/web/HttpServer.cpp` | feat(web): pines de buses editables + SD CS movido |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): pines de buses editables + SD CS movido |


## v1.93.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): reservar pines MCP configurados como in/out |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): reservar pines MCP configurados como in/out |


## v1.92.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/ConfigManager.cpp` | feat(web): General separado, SD CS dropdown, buses condicionales, demo MCP23S17 |
| `src/core/web/HttpServer.cpp` | feat(web): General separado, SD CS dropdown, buses condicionales, demo MCP23S17 |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): General separado, SD CS dropdown, buses condicionales, demo MCP23S17 |


## v1.91.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): MCP23S17 como expansor (pines en selectores) |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): MCP23S17 como expansor (pines en selectores); docs: tabla de pines por board (WROOM + S3) |


## v1.90.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): modales personalizados (reemplazan confirm/alert) |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): modales personalizados (reemplazan confirm/alert) |


## v1.89.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/ConfigManager.cpp` | fix(hw): pines reservados no aparecen + defaults sin solapar |
| `src/core/web/HttpServer.cpp` | fix(hw): pines reservados no aparecen + defaults sin solapar |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(hw): pines reservados no aparecen + defaults sin solapar |


## v1.88.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): CSS pulido + pines ocupados deshabilitados |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): CSS pulido + pines ocupados deshabilitados |


## v1.87.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): quitar pie de tarjeta (sensor_id/quality) |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): quitar pie de tarjeta (sensor_id/quality) |


## v1.86.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): nombres de tarjetas en espanol + mas traducciones |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): nombres de tarjetas en espanol + mas traducciones; docs: aclarar keys revocables (extra_keys) en PENDIENTES |


## v1.85.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): separar WiFi y mDNS + etiqueta temp |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): separar WiFi y mDNS + etiqueta temp |


## v1.84.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): agrandar JSON de /api/v1/system para datos WiFi |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): agrandar JSON de /api/v1/system para datos WiFi |


## v1.83.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `include/core/network/WiFiManager.hpp` | fix(web): temp ESP corregida, toggle mDNS y estilos |
| `src/core/SemaCore.cpp` | fix(web): temp ESP corregida, toggle mDNS y estilos |
| `src/core/network/WiFiManager.cpp` | fix(web): temp ESP corregida, toggle mDNS y estilos |
| `src/core/web/HttpServer.cpp` | fix(web): temp ESP corregida, toggle mDNS y estilos |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): temp ESP corregida, toggle mDNS y estilos |


## v1.82.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): ojo contrasena, config avanzada, datos WiFi, bus I2C unico |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): ojo contrasena, config avanzada, datos WiFi, bus I2C unico |


## v1.81.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `include/core/SemaCore.hpp` | feat(demo): magnitudes derivadas + datos ESP en sistema |
| `src/core/SemaCore.cpp` | feat(demo): magnitudes derivadas + datos ESP en sistema |
| `src/core/web/HttpServer.cpp` | feat(demo): magnitudes derivadas + datos ESP en sistema |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(demo): magnitudes derivadas + datos ESP en sistema |


## v1.80.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): selectores de pines GPIO (sin asignar + libres) |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): selectores de pines GPIO (sin asignar + libres) |


## v1.79.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(demo): datos ficticios mas completos (bateria, solar, pm10, rafagas) |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(demo): datos ficticios mas completos (bateria, solar, pm10, rafagas) |


## v1.78.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(demo): convertir valores al cambiar unidades |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(demo): convertir valores al cambiar unidades |


## v1.77.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): unificar nav y traducir titulos de seccion |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): unificar nav y traducir titulos de seccion |


## v1.76.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/ConfigManager.cpp` | fix(core): recuperar guardado cuando NVS esta lleno |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(core): recuperar guardado cuando NVS esta lleno; docs: retirar multi-idioma (PENDIENTES.md) |


## v1.75.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): multi-idioma es/en |
| `src/core/ConfigManager.cpp` | feat(web): multi-idioma es/en |
| `src/core/web/HttpServer.cpp` | feat(web): multi-idioma es/en |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): multi-idioma es/en |


## v1.74.0 — 2026-10-08

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): incluir kThemeJs en paginas de configuracion |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): incluir kThemeJs en paginas de configuracion |


## v1.73.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): selector de unidades metrico/imperial |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): selector de unidades metrico/imperial; docs: retirar paginas por seccion (PENDIENTES.md) |


## v1.72.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` | feat(web): paginas por seccion /sensors y /events |
| `src/core/web/HttpServer.cpp` | feat(web): paginas por seccion /sensors y /events |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): paginas por seccion /sensors y /events |


## v1.71.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | refactor(web): quitar seccion Salidas/entradas GPIO |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | refactor(web): quitar seccion Salidas/entradas GPIO; docs: retirar agregacion por niveles (PENDIENTES.md) |


## v1.70.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/storage/HistoryStore.hpp` | feat(storage): agregacion por niveles del historico |
| `src/core/SemaCore.cpp` | feat(storage): agregacion por niveles del historico |
| `src/core/storage/HistoryStore.cpp` | feat(storage): agregacion por niveles del historico |
| `src/core/web/HttpServer.cpp` | feat(storage): agregacion por niveles del historico |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(storage): agregacion por niveles del historico |


## v1.69.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/hw/HwProfile.hpp` | feat(hw): deshabilitar registros de desplazamiento por flag |
| `src/core/SemaCore.cpp` | feat(hw): deshabilitar registros de desplazamiento por flag |
| `src/core/web/HttpServer.cpp` | feat(hw): deshabilitar registros de desplazamiento por flag |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(hw): deshabilitar registros de desplazamiento por flag |


## v1.68.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(storage): microSD configurable + historico en SD |
| `include/core/storage/HistoryStore.hpp` | feat(storage): microSD configurable + historico en SD |
| `src/core/ConfigManager.cpp` | feat(storage): microSD configurable + historico en SD |
| `src/core/SemaCore.cpp` | feat(storage): microSD configurable + historico en SD |
| `src/core/storage/HistoryStore.cpp` | feat(storage): microSD configurable + historico en SD |
| `src/core/web/HttpServer.cpp` | feat(storage): microSD configurable + historico en SD |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(storage): microSD configurable + historico en SD; docs: retirar watchdog jerarquico (PENDIENTES.md) |


## v1.67.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/HealthMonitor.hpp` | feat(core): watchdog jerarquico por tarea |
| `src/core/HealthMonitor.cpp` | feat(core): watchdog jerarquico por tarea |
| `src/core/SemaCore.cpp` | feat(core): watchdog jerarquico por tarea |
| `src/core/web/HttpServer.cpp` | feat(core): watchdog jerarquico por tarea |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(core): watchdog jerarquico por tarea |


## v1.66.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/hw/HwProfile.hpp` | feat(web): pines de buses por board + SD SPI |
| `src/core/web/HttpServer.cpp` | feat(web): pines de buses por board + SD SPI |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): pines de buses por board + SD SPI; docs: retirar pendientes hechos (PENDIENTES.md) |


## v1.65.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/Time.hpp` | feat(core): zona horaria + config fail-safe |
| `src/core/ConfigManager.cpp` | feat(core): zona horaria + config fail-safe |
| `src/core/SemaCore.cpp` | feat(core): zona horaria + config fail-safe |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(core): zona horaria + config fail-safe |


## v1.64.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): registros en cascada + tooltips expansores |
| `include/core/ShiftRegisterManager.hpp` | feat(web): registros en cascada + tooltips expansores |
| `src/core/ConfigManager.cpp` | feat(web): registros en cascada + tooltips expansores |
| `src/core/SemaCore.cpp` | feat(web): registros en cascada + tooltips expansores |
| `src/core/ShiftRegisterManager.cpp` | feat(web): registros en cascada + tooltips expansores |
| `src/core/web/HttpServer.cpp` | feat(web): registros en cascada + tooltips expansores |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): registros en cascada + tooltips expansores |


## v1.63.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/hw/HwProfile.hpp` | feat(hw): pines RMII Ethernet oficiales documentados |
| `src/core/web/HttpServer.cpp` | feat(hw): pines RMII Ethernet oficiales documentados |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(hw): pines RMII Ethernet oficiales documentados |


## v1.62.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(hw): MCP23S17 (SPI) + registro con LATCH por chip |
| `include/core/ShiftRegisterManager.hpp` | feat(hw): MCP23S17 (SPI) + registro con LATCH por chip |
| `src/core/ConfigManager.cpp` | feat(hw): MCP23S17 (SPI) + registro con LATCH por chip |
| `src/core/ShiftRegisterManager.cpp` | feat(hw): MCP23S17 (SPI) + registro con LATCH por chip |
| `src/core/web/HttpServer.cpp` | feat(hw): MCP23S17 (SPI) + registro con LATCH por chip |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(hw): MCP23S17 (SPI) + registro con LATCH por chip |


## v1.61.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): expansores separados (MCP23017 + registro + GPIO) |
| `src/core/ConfigManager.cpp` | feat(web): expansores separados (MCP23017 + registro + GPIO) |
| `src/core/web/HttpServer.cpp` | feat(web): expansores separados (MCP23017 + registro + GPIO) |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): expansores separados (MCP23017 + registro + GPIO) |


## v1.60.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): canal ADS1115 desde sensor analogico |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): canal ADS1115 desde sensor analogico |


## v1.59.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): ADS1115 canal + magnitud |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): ADS1115 canal + magnitud |


## v1.58.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/Ds18b20Sensor.hpp` | feat(sensors): DS18B20 lectura por direccion ROM |
| `src/core/SemaCore.cpp` | feat(sensors): DS18B20 lectura por direccion ROM |
| `src/core/sensors/Ds18b20Sensor.cpp` | feat(sensors): DS18B20 lectura por direccion ROM |
| `src/core/sensors/SensorFactory.cpp` | feat(sensors): DS18B20 lectura por direccion ROM |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(sensors): DS18B20 lectura por direccion ROM |


## v1.57.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | fix(web): DS18B20 un solo pin + direccion ROM |
| `src/core/ConfigManager.cpp` | fix(web): DS18B20 un solo pin + direccion ROM |
| `src/core/web/HttpServer.cpp` | fix(web): DS18B20 un solo pin + direccion ROM |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): DS18B20 un solo pin + direccion ROM |


## v1.56.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): fuente ADC + multiples DS18B20 |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): fuente ADC + multiples DS18B20 |


## v1.55.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): sensores especificos (veleta, lluvia, viento, bateria) |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): sensores especificos (veleta, lluvia, viento, bateria) |


## v1.54.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): tooltips sensores + nombres claros |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): tooltips sensores + nombres claros |


## v1.53.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` | feat(web): fix switches + expansores/salidas |
| `src/core/web/HttpServer.cpp` | feat(web): fix switches + expansores/salidas |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): fix switches + expansores/salidas |


## v1.52.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | fix(web): sensores deshabilitados por defecto + direccion I2C |
| `src/core/ConfigManager.cpp` | fix(web): sensores deshabilitados por defecto + direccion I2C |
| `src/core/web/HttpServer.cpp` | fix(web): sensores deshabilitados por defecto + direccion I2C |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): sensores deshabilitados por defecto + direccion I2C |


## v1.51.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` | feat(web): config sensores + pines |
| `src/core/web/HttpServer.cpp` | feat(web): config sensores + pines |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): config sensores + pines |


## v1.50.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` | feat(web): dashboard publico + footer |
| `src/core/web/HttpServer.cpp` | feat(web): dashboard publico + footer |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): dashboard publico + footer |


## v1.49.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/ConfigManager.cpp` | fix(web): layout persistente + modo edicion + graficas |
| `src/core/web/HttpServer.cpp` | fix(web): layout persistente + modo edicion + graficas |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): layout persistente + modo edicion + graficas |


## v1.48.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | fix(web): mkey colision + layout NVS separado + historico demo |
| `include/core/web/HttpServer.hpp` | fix(web): mkey colision + layout NVS separado + historico demo |
| `src/core/ConfigManager.cpp` | fix(web): mkey colision + layout NVS separado + historico demo |
| `src/core/web/HttpServer.cpp` | fix(web): mkey colision + layout NVS separado + historico demo |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): mkey colision + layout NVS separado + historico demo |


## v1.47.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): dashboard Cargando (IIFE) + OTA progreso |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): dashboard Cargando (IIFE) + OTA progreso |


## v1.46.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): buffer JSON + pines en sensores |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): buffer JSON + pines en sensores |


## v1.45.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): PUT config webAuthed + tarjeta reloj NTP |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): PUT config webAuthed + tarjeta reloj NTP |


## v1.44.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` | feat(demo): modo demo + pagina sensores + mDNS + OTA manifest |
| `include/hw/HwProfile.hpp` | feat(demo): modo demo + pagina sensores + mDNS + OTA manifest |
| `platformio.ini` | feat(demo): modo demo + pagina sensores + mDNS + OTA manifest |
| `src/core/web/HttpServer.cpp` | feat(demo): modo demo + pagina sensores + mDNS + OTA manifest |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(demo): modo demo + pagina sensores + mDNS + OTA manifest |


## v1.43.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/network/WiFiManager.cpp` | feat(web): CSS unificado dashboard/config + mDNS saneado |
| `src/core/web/HttpServer.cpp` | feat(web): CSS unificado dashboard/config + mDNS saneado |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): CSS unificado dashboard/config + mDNS saneado |


## v1.42.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` | fix(web): auth unificada paginas config + NTP listado + dashboard |
| `src/core/web/HttpServer.cpp` | fix(web): auth unificada paginas config + NTP listado + dashboard |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): auth unificada paginas config + NTP listado + dashboard |


## v1.41.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): rutas /config/* + NTP + OTA check + dashboard limpio |
| `include/core/web/HttpServer.hpp` | feat(web): rutas /config/* + NTP + OTA check + dashboard limpio |
| `src/core/ConfigManager.cpp` | feat(web): rutas /config/* + NTP + OTA check + dashboard limpio |
| `src/core/SemaCore.cpp` | feat(web): rutas /config/* + NTP + OTA check + dashboard limpio |
| `src/core/web/HttpServer.cpp` | feat(web): rutas /config/* + NTP + OTA check + dashboard limpio |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): rutas /config/* + NTP + OTA check + dashboard limpio |


## v1.40.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): multi-pagina + IP estatica + claves API |
| `include/core/network/WiFiManager.hpp` | feat(web): multi-pagina + IP estatica + claves API |
| `include/core/web/HttpServer.hpp` | feat(web): multi-pagina + IP estatica + claves API |
| `src/core/ConfigManager.cpp` | feat(web): multi-pagina + IP estatica + claves API |
| `src/core/SemaCore.cpp` | feat(web): multi-pagina + IP estatica + claves API |
| `src/core/network/WiFiManager.cpp` | feat(web): multi-pagina + IP estatica + claves API |
| `src/core/web/HttpServer.cpp` | feat(web): multi-pagina + IP estatica + claves API |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): multi-pagina + IP estatica + claves API |


## v1.39.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): sin parpadeo + escaneo WiFi robusto |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): sin parpadeo + escaneo WiFi robusto |


## v1.38.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | fix(web): gridstack sin superposicion (removeAll + compact) |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(web): gridstack sin superposicion (removeAll + compact) |


## v1.37.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/ConfigManager.hpp` | feat(web): escaneo WiFi, login usuario/pass, navbar, auto-reinicio |
| `include/core/web/HttpServer.hpp` | feat(web): escaneo WiFi, login usuario/pass, navbar, auto-reinicio |
| `src/core/ConfigManager.cpp` | feat(web): escaneo WiFi, login usuario/pass, navbar, auto-reinicio |
| `src/core/web/HttpServer.cpp` | feat(web): escaneo WiFi, login usuario/pass, navbar, auto-reinicio |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): escaneo WiFi, login usuario/pass, navbar, auto-reinicio |


## v1.36.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` | fix(config): buffer JSON 16KB + red WiFi separada |
| `src/core/ConfigManager.cpp` | fix(config): buffer JSON 16KB + red WiFi separada |
| `src/core/web/HttpServer.cpp` | fix(config): buffer JSON 16KB + red WiFi separada |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | fix(config): buffer JSON 16KB + red WiFi separada |


## v1.35.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` | feat(ota): verificacion de integridad por SHA-256 |
| `src/core/web/HttpServer.cpp` | feat(ota): verificacion de integridad por SHA-256 |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(ota): verificacion de integridad por SHA-256 |


## v1.34.0 — 2026-10-07

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | feat(web): tema claro/oscuro persistente |
| `CHANGELOG.md` · `firmware_manifest.json` · `include/core/Version.hpp` · `docs/*` | feat(web): tema claro/oscuro persistente |

## v1.33.0

| Archivo | Cambio |
|---------|--------|
| `src/core/storage/HistoryStore.*` | retención por tiempo (`prune`) |
| `src/core/web/HttpServer.cpp` | `/api/v1/history?format=csv` + botón CSV |
| `src/core/SemaCore.cpp` | poda periódica del histórico |

## v1.32.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | gráficos con series, ejes y rangos |
| `docs/ETHERNET-Y-BUILDFLAGS.md` | documentación Ethernet + build_flags |

## v1.31.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | catálogo de tarjetas + añadir/eliminar + gráficos |

## v1.30.0

| Archivo | Cambio |
|---------|--------|
| `data/gridstack-all.min.js` + `gridstack.min.css` | Gridstack servido desde LittleFS |
| `src/core/web/HttpServer.*` | rutas `/gridstack.min.css` y `/gridstack-all.min.js` |
| `platformio.ini` | `board_build.filesystem = littlefs` |

## v1.29.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | dashboard Gridstack + endpoints `/wind/resistors` y `/dashboard/layout` |
| `src/core/derived/DerivedCalculator.cpp` | veleta por tabla de 8 resistencias |
| `ConfigManager` | `system.wind_resistors[]`, `wind_rpull`, `dashboard_layout` |

## v1.28.0

| Archivo | Cambio |
|---------|--------|
| `include/core/derived/DerivedCalculator.hpp` + `.cpp` | magnitudes derivadas + unidades |
| `include/core/ConfigManager.hpp` + `.cpp` | `system.units`/`altitude`/veleta + `sensors[].enabled` |
| `src/core/web/HttpServer.*` | `/api/v1/sensors` (derivadas) + `/api/v1/wind/north` |

## v1.27.0

| Archivo | Cambio |
|---------|--------|
| `firmware_manifest.json` | multi-chip (un binario + SHA por board) |
| `partitions_{4,8,16}mb.csv` | tablas de particiones por tamaño de flash |
| `platformio.ini` | env `esp32-wroom-32u` (16 MB) + particiones por env |
| `HwProfile.hpp` | `SEMA_BOARD_ID` + `SEMA_FLASH_MB` + `BOARD_ESP32_WROOM32U` |

## v1.26.0

| Archivo | Cambio |
|---------|--------|
| `include/hw/HwProfile.hpp` | perfil de hardware (board + features + pines) |
| `platformio.ini` | `build_flags` de features y variantes |
| `SemaCore`/`HttpServer`/managers | compilación condicional `#if SEMA_USE_*` |

## v1.25.0

| Archivo | Cambio |
|---------|--------|
| `src/core/EthernetManager.cpp` | W5500 vía driver ESP-IDF `esp_eth` (integrado a lwIP) |
| `include/core/ConfigManager.hpp` + `.cpp` | campos `irq`/`sck`/`miso`/`mosi` |

## v1.24.0

| Archivo | Cambio |
|---------|--------|
| `include/core/EthernetManager.hpp` + `.cpp` | Ethernet LAN8720A (nativo) / W5500 (SPI) |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `ethernet` |
| `platformio.ini` | `[env:base]` + `build_flags` (`BOARD_ESP32_WROOM`/`BOARD_ESP32_S3`) |

## v1.23.0

| Archivo | Cambio |
|---------|--------|
| `include/core/ZigbeeManager.hpp` + `.cpp` | nuevo Zigbee (ZNP por UART) |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `zigbee` |
| `src/core/web/HttpServer.*` | `/api/v1/zigbee` GET/POST |

## v1.22.0

| Archivo | Cambio |
|---------|--------|
| `include/core/LoraManager.hpp` + `.cpp` | nuevo LoRa SX1262 (RadioLib) |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `lora` |
| `src/core/web/HttpServer.*` | `/api/v1/lora` GET/POST |

## v1.21.0

| Archivo | Cambio |
|---------|--------|
| `include/core/CanManager.hpp` + `.cpp` | nuevo CAN/TWAI |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `can` |
| `src/core/web/HttpServer.*` | `/api/v1/can` GET/POST |

## v1.20.0

| Archivo | Cambio |
|---------|--------|
| `include/core/ModbusManager.hpp` + `.cpp` | nuevo maestro Modbus RTU |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `modbus` |
| `src/core/web/HttpServer.*` | `/api/v1/modbus` |

## v1.19.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/SolarSensor.hpp` + `.cpp` | nuevo driver SOLAR |
| `src/core/sensors/SensorFactory.cpp` | registro `"SOLAR"` |

## v1.18.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/CoSensor.hpp` + `.cpp` | nuevo driver CO |
| `src/core/sensors/SensorFactory.cpp` | registro `"CO"` |

## v1.17.0

| Archivo | Cambio |
|---------|--------|
| `include/core/ShiftRegisterManager.hpp` + `.cpp` | nuevo (74HC595/74HC165) |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `shift_register` |
| `src/core/web/HttpServer.*` | `/api/v1/shift` GET/POST |

## v1.16.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | Cookie `SameSite=Strict` en login/logout |
| `include/core/Version.hpp` | `1.16.0` |
| `firmware_manifest.json`, `CHANGELOG.md`, `docs/*` | versión y docs |

## v1.15.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/Ads1115Sensor.hpp` + `.cpp` | nuevo driver ADS1115 |
| `src/core/sensors/SensorFactory.cpp` | registro `"ADS1115"` |
| `platformio.ini` | librería ADS1X15 |

## v1.14.0

| Archivo | Cambio |
|---------|--------|
| `include/core/GpioManager.hpp` + `.cpp` | soporte MCP23017 (`expander_addr`) |
| `include/core/ConfigManager.hpp` + `.cpp` | campo `expander_addr` en `GpioSpec` |
| `platformio.ini` | librería MCP23017 |

## v1.13.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/As3935Sensor.hpp` + `.cpp` | nuevo driver AS3935 |
| `src/core/sensors/SensorFactory.cpp` | registro `"AS3935"` |
| `platformio.ini` | librería SparkFun AS3935 |

## v1.12.0

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` + `.cpp` | expiración de sesión (1 h) |

## v1.11.0

| Archivo | Cambio |
|---------|--------|
| `include/core/publishers/MqttPublisher.hpp` + `.cpp` | `configure(..., user, pass)` |
| `include/core/ConfigManager.hpp` + `.cpp` | `mqtt_user` / `mqtt_pass` |

## v1.10.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/Sgp30Sensor.hpp` + `.cpp` | nuevo driver SGP30 |
| `platformio.ini` | librería SGP30 |

## v1.9.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/SensorManager.hpp` | `clear()` |
| `src/core/SemaCore.cpp` + `.hpp` | `applySensors()` (hot reload de sensores) |

## v1.8.0

| Archivo | Cambio |
|---------|--------|
| `include/core/publishers/HttpPublisher.hpp` / `MqttPublisher.hpp` | setters de config |
| `src/core/SemaCore.cpp` + `.hpp` | `applyPublishers()` + publishers como miembros |

## v1.7.0

| Archivo | Cambio |
|---------|--------|
| `include/core/GpioManager.hpp` + `.cpp` | nuevo (GPIO standalone) |
| `include/core/ConfigManager.hpp` + `.cpp` | `gpio[]` |
| `src/core/web/HttpServer.*` | `/api/v1/gpio` GET/POST |

## v1.6.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | rate limiting en login |

---

## Ver también

- [CHANGELOG](CHANGELOG.md) · [Evolución](Evolucion.md) · [Mejoras y roadmap](Mejoras-y-roadmap.md)
- [Referencia de código](Referencia-de-codigo.md) · [Guía de desarrollo](Guia-de-desarrollo.md) · [Home](Home.md)
