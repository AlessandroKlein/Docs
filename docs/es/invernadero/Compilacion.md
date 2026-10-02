# Compilación y flasheo

> **Tipo:** Embebidos | **Estado:** Estable | **Fecha:** 2026-10-02

```bash
pio run              # compila el entorno por defecto (esp32doit-devkit-v1)
pio run -e esp32-s3-devkitc-1   # compila otro target
pio run -t upload    # graba por puerto serie
pio device monitor   # monitor serie (115200)
```

## Multi-board

`platformio.ini` define entornos para ESP32 / S2 / S3 / C3 / C6.

## Particiones

`board_build.partitions = partitions/default_8MB.csv` (8 MB, por defecto).
Alternativas: `default.csv` (4 MB), `default_16MB.csv` (16 MB).

## Memoria (v3.13.0)

RAM ~27 % · Flash ~38 % (partición de 8 MB).

## Verificación

- `pio run` termina en `SUCCESS`.
- SHA-256 del binario en `firmware_manifest.json`.
