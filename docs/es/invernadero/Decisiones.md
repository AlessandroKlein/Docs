# Decisiones de arquitectura

> **Tipo:** Referencia | **Estado:** Cerrado | **Fecha:** 2026-10-02

Documento fuente: [`docs/DUDAS-Y-DECISIONES.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/DUDAS-Y-DECISIONES.md).

## Principio rector

> El servidor administra y coordina. El ESP32 controla y protege.

## Resumen de decisiones

1. **Retención de históricos**: PostgreSQL particionado por mes + agregación progresiva.
2. **Variables calculadas (VPD)**: cálculo local (automatización) + central (históricos).
3. **Ubicación**: `latitude/longitude/elevation/timezone` por invernadero.
4. **Estación meteorológica**: `WeatherManager` con adaptadores (Open-Meteo/REST/MQTT/Modbus).
5. **AP**: SSID único + contraseña aleatoria/persistente (no derivada de MAC).
6. **Web local**: autenticación obligatoria + hardware avanzado protegido.
7. **TLS**: preparado en V8; obligatorio para comunicación remota.
8. **MQTT provisioning**: DISCOVERED → PENDING → COMMISSIONED → ACTIVE (+ BLOCKED/REVOKED).
9. **Device Shadow**: `desired` / `reported` / `actual`.
10. **Identidad**: `device_id`/`device_uid`/`hardware_id`/`mac_address`.
11. **PCNT**: `PulseCounterManager` para caudal/lluvia/anemómetro.
12. **Modbus**: sin autodiscovery universal; perfiles + commissioning.
13. **CAN/TWAI**: `CanManager` + transceptor externo SN65HVD23X.
14. **Sistema de módulos**: `ModuleRegistry` en firmware/servidor/frontend.
15. **Token de API**: persistente en NVS para control desde el servidor central.

## Invariantes

1. El dispositivo funciona sin servidor.
2. El servidor nunca es necesario para una función automática crítica.
3. La automatización se ejecuta localmente.
4. Las comunicaciones son una capa independiente de la lógica de control.
5. El servidor administra; el ESP32 mide, decide, controla y protege.
