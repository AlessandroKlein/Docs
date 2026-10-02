# Inicio rápido

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-02

Guía para poner en marcha el sistema en minutos.

## 1. Conectar el hardware

1. Conectar el ESP32 a la fuente (5 V).
2. Conectar sensores/actuadores según [Hardware y conexiones](Hardware-y-Conexiones.md)
   (respetando pines y pull-ups).

## 2. Primer arranque

- El ESP32 arranca un **Access Point** `INVERNADERO-XXXXXX` (derivado de la MAC).
- Conectarse a esa red y abrir `http://invernadero.local` (o `http://192.168.4.1`).

## 3. Configurar WiFi

1. En la web, ir a configuración de red.
2. Elegir WiFi (`net_interface: wifi`) y cargar SSID + contraseña.
3. (Opcional) elegir Ethernet (W5500) con `net_interface: ethernet` + pin CS.

## 4. Acceder y configurar

- Desde la red local: `http://invernadero.local`.
- Configurar sensores, actuadores, umbrales y reglas desde la web o `PUT /api/v1/config`.

## 5. Ver datos y controlar

- Dashboard local (`/`) con sensores, actuadores y alarmas.
- Página de pines (`/pins`) y autodetección (`/api/v1/detect`).
- API REST completa en [API REST](API-REST.md).

## 6. Conectar al servidor central (opcional)

1. Configurar MQTT (`mqtt_host`, etc.).
2. Generar el **token de API** en la sección "API / Servidor central".
3. Registrar el dispositivo en el servidor (provisioning).

## 7. OTA

Actualizar por WiFi (ArduinoOTA) o desde el servidor central (ver [OTA](OTA-y-Actualizacion.md)).
