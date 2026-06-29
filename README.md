# MLOps4RT-Edge

Pipeline MLOps por fases para llevar datos temporales hasta modelos cuantizados y validación en edge.

El repositorio contiene **código y automatización**. Los datos, ejecuciones, modelos, logs, cachés DVC y estado de MLflow son artefactos locales del proyecto y no deben versionarse aquí.

## Qué Hace

El flujo completo tiene ocho fases:

| Fase | Objetivo | Entrada principal | Salida principal |
| --- | --- | --- | --- |
| F01 | Explorar y limpiar datos | dataset bruto | dataset limpio |
| F02 | Convertir señales en eventos | F01 | catálogo/eventos |
| F03 | Crear ventanas temporales | F02 | ventanas de eventos |
| F04 | Crear targets de predicción | F03 | dataset supervisado |
| F05 | Entrenar modelos | F04 | modelo entrenado |
| F06 | Cuantizar y empaquetar | F05 | modelo TFLite/edge |
| F07 | Validar un modelo en edge | F06 | métricas de runtime |
| F08 | Validar un sistema multimodelo | F07 | configuración y métricas globales |

Cada fase trabaja con **variantes**. Una variante queda en:

```text
executions/<fase>/<variante>/
```

Ejemplo:

```text
executions/f05_modeling/v5_0001/
```

## Requisitos

Mínimos:

- Python 3.11
- GNU Make
- Git

Recomendados:

- Docker, necesario para F05/F06 y flujos ESP32 reproducibles.
- DVC, si se registran artefactos pesados.
- MLflow, si se quiere tracking de entrenamiento.
- Toolchain o placa edge para F07/F08 físicos.

En Windows se recomienda ejecutar `make` desde Git Bash.

## Setup

Modo local:

```bash
make setup SETUP_CFG=setup/local.yaml
make check-setup
```

Modo remoto:

```bash
cp setup/remote.yaml .mlops4ofp.remote.yaml
# editar endpoints Git/DVC/MLflow
make setup SETUP_CFG=.mlops4ofp.remote.yaml
make check-setup
```

Ver ayuda completa:

```bash
make help
```

Limpiar setup local:

```bash
make clean-setup
```

## Patrón De Uso

Para cada fase:

```bash
make variantN ...
make scriptN VARIANT=vN_XXXX
make checkN VARIANT=vN_XXXX
make registerN VARIANT=vN_XXXX
```

Ejemplo para F01:

```bash
make variant1 VARIANT=v1_0001 RAW=data/raw.csv CLEANING=basic
make script1 VARIANT=v1_0001
make check1 VARIANT=v1_0001
make register1 VARIANT=v1_0001
```

Los IDs cortos también se aceptan. Por ejemplo `v001` se normaliza a la fase correspondiente.

## Ejecución Completa Por Fases

### F01: Datos Limpios

```bash
make variant1 VARIANT=v1_0001 RAW=data/raw.csv CLEANING=basic NAN_VALUES='[-999999]'
make script1 VARIANT=v1_0001
make check1 VARIANT=v1_0001
make register1 VARIANT=v1_0001
```



Limpieza profunda FO1:
#### Limpieza profunda F01:
```bash
make variant1 VARIANT=v000 RAW=data/raw.csv CLEANING=basic NAN_VALUES='[-999999]' ERROR_VALUES='{"MG-LV-MSB_AC_Voltage":[0.0],"Receiving_Point_AC_Voltage":[0.0],"Island_mode_MCCB_AC_Voltage":[0.0],"Island_mode_MCCB_Frequency":[-327.679993,0.0],"MG-LV-MSB_Frequency":[-327.679993,0.0],"Outlet_Temperature":[-52.5],"Inlet_Temperature_of_Chilled_Water":[-52.5,-52.400002,-52.299999]}'


### F02: Eventos

```bash
make variant2 VARIANT=v2_0001 PARENT=v1_0001 STRATEGY=transitions BANDS='[10,50,90]' NAN_MODE=discard
make script2 VARIANT=v2_0001
make check2 VARIANT=v2_0001
make register2 VARIANT=v2_0001
```

### F03: Ventanas

```bash
make variant3 VARIANT=v3_0001 PARENT=v2_0001 OW=60 LT=10 PW=10 STRATEGY=synchro NAN_MODE=discard
make script3 VARIANT=v3_0001
make check3 VARIANT=v3_0001
make register3 VARIANT=v3_0001
```

### F04: Targets

```bash
make variant4 VARIANT=v4_0001 PARENT=v3_0001 NAME=target_name OPERATOR=OR EVENTS='["event_a","event_b"]'
make script4 VARIANT=v4_0001
make check4 VARIANT=v4_0001
make register4 VARIANT=v4_0001
```

### F05: Entrenamiento

```bash
make variant5 VARIANT=v5_0001 PARENT=v4_0001 MODEL_FAMILY=c1nnd IMBALANCE_STRATEGY=rare_events
make script5 VARIANT=v5_0001
make check5 VARIANT=v5_0001
make register5 VARIANT=v5_0001
```

F05 corre en Docker. Para GPU:

```bash
make script5 VARIANT=v5_0001 F56_GPU=true
```

### F06: Cuantización

```bash
make variant6 VARIANT=v6_0001 PARENT=v5_0001 DEPLOY_TARGET=esp32 REQUIRE_INT8=false
make script6 VARIANT=v6_0001
make check6 VARIANT=v6_0001
make register6 VARIANT=v6_0001
```

Para GPU:

```bash
make script6 VARIANT=v6_0001 F56_GPU=true
```

### F07: Validación De Un Modelo En Edge

Virtual ESP32:

```bash
make variant7 VARIANT=v7_0001 PARENT=v6_0001 PLATFORM=esp32 MTI_MS=100 TIME_SCALE=0.01 VIRTUAL=true MAX_ROWS=10000 ESP_FLASH_MB=4
make script7 VARIANT=v7_0001
make check7 VARIANT=v7_0001
make register7 VARIANT=v7_0001
```

Placa física:

```bash
make variant7 VARIANT=v7_0002 PARENT=v6_0001 PLATFORM=esp32 MTI_MS=100 TIME_SCALE=0.01 VIRTUAL=false MAX_ROWS=10000 ESP_FLASH_MB=4

make script7 VARIANT=v7_0002 
```

En Windows usa el puerto real, por ejemplo:

```bash
make script7 VARIANT=v7_0002 
```

Ejecución paso a paso:

```bash
make script7-prepare-build VARIANT=v7_0001
make script7-flash-run VARIANT=v7_0001
make script7-post VARIANT=v7_0001
```

Opciones útiles:

- `MTI_MS`: presupuesto temporal de inferencia en milisegundos.
- `TIME_SCALE`: escala temporal usada para reproducir ventanas en edge.
- `MAX_ROWS`: limita filas incluidas en el firmware/dataset generado.
- `MAX_LINES`: limita líneas enviadas por serial, sin recortar el dataset generado.
- `ESP_FLASH_MB`: tamaño de flash ESP32 en MB. Valores típicos: `2`, `4`, `8`, `16`. Úsalo si el firmware no cabe en la partición por defecto o si tu placa/QEMU declara más de 2 MB.
- `F07_FORCE_REBUILD=true`: recompila aunque exista build previo.

### F08: Validación Multimodelo

```bash
make variant8 VARIANT=v8_0001 PARENTS=v7_0001,v7_0002 PLATFORM=esp32 MTI_MS=100
make script8 VARIANT=v8_0001
make check8 VARIANT=v8_0001
make register8 VARIANT=v8_0001
```

Selección automática por ILP:

```bash
make variant8 VARIANT=v8_0002 PARENTS=v7_0001,v7_0002 PLATFORM=esp32 MTI_MS=100 \
  SELECTION_MODE=auto_ilp OBJECTIVE=max_global_recall
```

## ESP32 Virtual

Si una variante F07/F08 tiene `VIRTUAL=true`, `make script7` o `make script8` usa el entorno virtual ESP32 basado en Docker, QEMU y `socat`.

Comandos útiles:

```bash
make esp32-virt-verify
make script7-virtualESP32 VARIANT=v7_0001
make esp32-virt-stop
```

El host solo necesita Docker. El entorno de ESP-IDF/QEMU vive dentro del contenedor.

## ESP32 Física

Para placa real:

1. Crear variante con `VIRTUAL=false`.
2. Conectar la placa.
4. Revisar logs en la carpeta de la variante.

Ejemplo:

```bash
make script7 VARIANT=v7_0002 PORT=/dev/ttyUSB0 BAUD=115200
```

Si una ejecución falla:

- Revisa `07_esp_flash_log.txt`.
- Revisa `07_esp_monitor_log.txt`.
- Revisa `metrics_models.csv`.
- Revisa `07_model_profile.yaml`.

Un `low_ok_rate` con muchos `wd_late` o `urgent` suele indicar que algunas inferencias no terminan dentro de `MTI_MS`.

## Artefactos

El pipeline genera:

- `params.yaml`: parámetros de la variante.
- `outputs.yaml`: salidas registrables de la fase.
- `metadata.yaml`: estado de ciclo de vida.
- `metrics_*.csv`: métricas.
- logs de build/flash/runtime en fases edge.
- modelos y datasets intermedios.

Estos archivos viven bajo `executions/`. No forman parte del código fuente.

## DVC Y MLflow

Responsabilidades:

- Git: código, documentación, schemas y templates.
- DVC: artefactos pesados.
- MLflow: tracking de experimentos.
- `executions/`: estado local de variantes.

Traer artefactos ya registrados:

```bash
make dvc-pull VARIANT=v5_0001
```

Traer varios:

```bash
make dvc-pull VARIANT=v2_0001,v5_0001,v7_0001
```

Limpiar artefactos descargados:

```bash
make dvc-clean VARIANT=v5_0001
```

## Limpieza

Eliminar una variante:

```bash
make remove5 VARIANT=v5_0001
```

Eliminar todas las variantes de una fase:

```bash
make remove5-all
```

Resetear el entorno local:

```bash
make clean-setup
```

## Troubleshooting

### `make check-setup` falla

Comprueba:

- Python 3.11 disponible.
- `.venv` creado.
- configuración en `.mlops4ofp/setup.yaml`.
- acceso a DVC/MLflow si están habilitados.

### Una fase no encuentra el parent

Comprueba que el parent existe y está en la fase correcta:

```text
executions/f0X_<phase>/<variant>/
```

Ejemplo: F05 debe apuntar a una variante F04.

### F05/F06 falla en Docker

Comprueba:

- Docker arrancado.
- imagen construible.
- espacio en disco.
- `F56_GPU=true` solo si tienes GPU y runtime NVIDIA configurado.

### F07/F08 no flashea

Comprueba:

- `PORT`.
- permisos del puerto serie.
- placa en modo correcto.
- `07_esp_flash_log.txt`.

### Firmware ESP32 demasiado grande

Prueba con flash mayor si tu placa lo soporta:

```bash
make variant7 VARIANT=v7_0003 PARENT=v6_0001 PLATFORM=esp32 MTI_MS=100 ESP_FLASH_MB=4
```

Si la placa solo tiene 2 MB, reduce modelo/operadores/trazas o cambia de hardware.

### Inferencias edge fallan a veces

Mira:

- `metrics_models.csv`
- `metrics_inference_records.csv`, si existe.
- `07_esp_monitor_log.txt`
- `phase_status_reason` en `outputs.yaml`

Indicadores comunes:

- `wd_late`: la inferencia empezó pero no terminó antes del deadline.
- `urgent`: fallback de emergencia por deadline.
- `inference_incomplete`: hubo inicio de inferencia sin fin registrado.
- `no_successful_inferences`: no hubo inferencias válidas.

## Estructura Del Repositorio

```text
scripts/core/              lógica común
scripts/phases/            fases F01-F08
scripts/runtime_analysis/  parser y métricas runtime
edge/                      templates y runtime edge
setup/                     configuración local/remota
test/                      auditorías y experimentos
executions/                salidas locales generadas
```

## Para Desarrolladores

Lee [DEVELOPERS.md](DEVELOPERS.md).

Antes de publicar cambios:

- no subir `executions/`;
- no subir `.env`;
- no subir cachés DVC/MLflow;
- actualizar este README si cambia el uso del pipeline.
