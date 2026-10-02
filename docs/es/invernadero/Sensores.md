# Sensores

> **Tipo:** Embebidos | **Estado:** Estable | **Fecha:** 2026-10-02

Todos los sensores son opcionales y se habilitan por configuración. Un sensor no
instalado se distingue de uno con error: **NO INSTALADO / DESHABILITADO /
SIN COMUNICACIÓN / ERROR / VÁLIDO / FUERA DE RANGO**.

## 1. Temperatura y humedad (SHT31 / AHT20)

- **SHT31**: I²C (0x44), ±0,2 °C / ±2 %RH.
- **AHT20**: I²C (0x38), alternativa económica.
- Abstracción `TempHumSensor` → las reglas no dependen del modelo físico.

## 2. DS18B20 (1-Wire)

- Bus en GPIO 4 con una resistencia de 4,7 kΩ común (a 3,3 V).
- Múltiples sensores (agua, tanque, sustrato, exterior).
- ±0,5 °C entre -10 y 85 °C.

## 3. Humedad de suelo (capacitivo + ADS1115)

- ADS1115: 16 bits, 4 canales (A0..A3 → zona 1..4).
- Calibración por zona (seco/húmedo), no es % universal.

## 4. Iluminación (BH1750)

- I²C (0x23), lux. Se diferencia lux de PPFD.

## 5. CO₂ (SCD40/SCD41)

- I²C (0x62), NDIR. Se activa/desactiva toda la lógica relacionada.

## 6. Nivel de depósito

- Continuo: ultrasónico (TRIG 25, ECHO 26).
- Seguridad: flotadores FLOAT_LOW (32), FLOAT_HIGH (33).

## 7. Caudal

- Caudalímetro por pulsos (GPIO 34, optoacoplado a 3,3 V).
- L/min, L/h y litros acumulados. Protege la bomba.

## 8. Lluvia y viento

- Pluviómetro (GPIO 35) y anemómetro (GPIO 36).
- Usados por el techo (cierre por lluvia/viento).

## 9. pH

- Analógico (ADS1115) con calibración 4/7/10, o Modbus (RS485).

## 10. EC (conductividad)

- Prioriza Modbus RTU; admite entrada analógica ADS1115.

## 11. Detección automática

El firmware detecta SHT31, ADS1115, MCP23017, SCD41 y DS18B20. La detección no
habilita funciones peligrosas: la activación requiere configuración.
