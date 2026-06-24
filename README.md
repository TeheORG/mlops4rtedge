# MLOps4RT-Edge

![Python 3.11](https://img.shields.io/badge/python-3.11-blue)
![Platforms macOS Linux Windows](https://img.shields.io/badge/platforms-macOS%20%7C%20Linux%20%7C%20Windows-success)
![Repository scope code only](https://img.shields.io/badge/repository-code--only-informational)
![Artifacts DVC backend](https://img.shields.io/badge/artifacts-DVC%20backend-orange)

MLOps4RT-Edge is a reproducible, phase-based MLOps pipeline for edge machine learning workflows, from raw time-series data to quantized models and on-device validation.

It is intended for teams that need a structured and traceable path from data preparation to embedded deployment experiments, while keeping binary artifacts and execution state outside the public code repository.

This README is written for users who want to run the pipeline with their own data, generate their own execution artifacts, and store those artifacts in their own project storage backends.

If you want to work on the pipeline code itself, see [DEVELOPERS.md](DEVELOPERS.md).

## At A Glance

1. Eight-phase pipeline from exploration to edge system validation.
2. Setup-driven project configuration for Git, DVC, and MLflow backends.
3. Cross-platform workflow for macOS, Linux, and Windows.
4. Public repository kept code-only by design.
5. Intended for project workspaces that keep executions and artifacts outside this repo.

## Project Status

This project is intended to be published as a reusable public pipeline codebase.

Current status:

1. The repository is maintained as a code-only public repository.
2. Generated executions, local DVC state, MLflow state, and project artifacts are intentionally excluded from Git.
3. Users are expected to run the pipeline in their own project workspace and connect it to their own DVC and MLflow backends.

## What This Project Does

The pipeline is organized into eight phases:

1. Explore raw data.
2. Build an events dataset.
3. Build a windows dataset.
4. Engineer prediction targets.
5. Train models.
6. Quantize and package models for edge deployment.
7. Validate a model on edge hardware.
8. Validate a multi-model system on edge hardware.

Each phase creates a variant under a local executions workspace and can be validated, registered, and removed independently.

## Repository Scope

This public repository is the pipeline codebase only.

- It does contain: source code, setup templates, tests, documentation, and automation.
- It does not contain: your local DVC cache, your MLflow state, your execution outputs, or your project-specific DVC references.

When you run the pipeline for your own project:

- Your generated outputs are written to your local workspace under executions/.
- Large binary artifacts must go to the DVC backend configured by your project setup.
- Your project Git repository, if used, is the repository defined in your setup, not this public code repository.

## Prerequisites

Minimum prerequisites:

1. Python 3.11.
2. GNU Make.
3. Git.
4. A working virtual environment can be created locally by the setup flow.

Optional but commonly needed:

1. Docker for containerized phases such as F06 and embedded build flows.
2. A supported edge platform toolchain and hardware for F07 and F08.
3. DVC and MLflow backends, configured through the project setup.

## Platform Compatibility

The pipeline is designed to be cross-platform at the project level and should be usable from macOS, Linux, and Windows.

What is platform-independent:

1. The phase model.
2. The Makefile-based workflow.
3. Setup-driven DVC and MLflow configuration.
4. Variant creation, validation, traceability, and registration logic.

What may still vary by operating system in practice:

1. Serial port names, such as `/dev/ttyUSB0` on Linux-like systems and `COMx` on Windows.
2. Serial port permissions on systems that protect device access.
3. Docker path handling, especially on Windows shells.
4. Vendor-specific toolchain installation details for embedded targets.

These are operational differences, not pipeline design differences.

## Quick Start

### 1. Clone the pipeline code

```bash
git clone https://github.com/STRAST-UPM/mlops4rtedge.git
cd mlops4rtedge
```

### 2. Choose a setup mode

For a fully local project workspace:

```bash
make setup SETUP_CFG=setup/local.yaml
make check-setup
```

For a project that should publish to its own remote services, start from the remote template:

```bash
cp setup/remote.yaml .mlops4ofp.remote.yaml
# edit the file with your own Git, DVC, and MLflow endpoints
make setup SETUP_CFG=.mlops4ofp.remote.yaml
make check-setup
```

The local setup template uses:

- local DVC storage at ./.dvc_storage
- local MLflow tracking at file:./mlruns
- no Git publishing from pipeline register commands

## Recommended Usage Model

For project work, use this repository as the pipeline engine and keep your data and executions in your own working copy or project repository.

Recommended pattern:

1. Clone this repository.
2. Run setup with either local or remote configuration.
3. Place your raw dataset in data/.
4. Create and execute variants phase by phase.
5. Store binary artifacts in the DVC backend configured by your project.
6. Keep any project-specific execution history in your own project environment.

## Common Commands

Show the built-in help:

```bash
make help
```

Set up the environment:

```bash
make setup SETUP_CFG=setup/local.yaml
make check-setup
```

Reset the local project environment:

```bash
make clean-setup
```

Remove all variants from a phase:

```bash
make remove-phase-all PHASE=f05_modeling VARIANTS_DIR=executions/f05_modeling
```

## Phase-by-Phase Example

### F01: Explore raw data

```bash
make variant1 VARIANT=v001 RAW=./data/raw.csv CLEANING=basic NAN_VALUES='[-999999]'
make script1 VARIANT=v001
make check1 VARIANT=v001
make register1 VARIANT=v001
```

#### Limpieza profunda F01:
```bash
make variant1 VARIANT=v000 RAW=data/raw.csv CLEANING=basic NAN_VALUES='[-999999]' ERROR_VALUES='{"MG-LV-MSB_AC_Voltage":[0.0],"Receiving_Point_AC_Voltage":[0.0],"Island_mode_MCCB_AC_Voltage":[0.0],"Island_mode_MCCB_Frequency":[-327.679993,0.0],"MG-LV-MSB_Frequency":[-327.679993,0.0],"Outlet_Temperature":[-52.5],"Inlet_Temperature_of_Chilled_Water":[-52.5,-52.400002,-52.299999]}'
make script1 VARIANT=v000
make check1 VARIANT=v000
make register1 VARIANT=v000
```




### F02: Build events dataset

```bash
# make variant2 VARIANT=v201 PARENT=v001 STRATEGY=levels BANDS='[0.1, 0.2, 0.3]' NAN_MODE=discard
make variant2 VARIANT=v201 PARENT=v001 STRATEGY=transitions BANDS='[10, 90]' NAN_MODE=discard
make script2 VARIANT=v201
make check2 VARIANT=v201
make register2 VARIANT=v201
```

### F03: Build windows dataset

```bash
make variant3 VARIANT=v301 PARENT=v201 OW=600 LT=100 PW=100 STRATEGY=synchro NAN_MODE=discard
make script3 VARIANT=v301
make check3 VARIANT=v301
make register3 VARIANT=v301
```

### F04: Create prediction targets

```bash
make variant4 VARIANT=v401 PARENT=v301 NAME=battery_overheat OPERATOR=OR EVENTS='["Battery_Active_Power_0_10-to-90_100,Battery_Active_Power_10_90-to-90_100"]'
make script4 VARIANT=v401
make check4 VARIANT=v401
make register4 VARIANT=v401
```

### F05: Train models

```bash
make variant5 VARIANT=v501 PARENT=v401 MODEL_FAMILY=dense_bow IMBALANCE_STRATEGY=none
make script5 VARIANT=v501
make check5 VARIANT=v501
make register5 VARIANT=v501
```

Common F05 overrides include batch size, epochs, learning rate, embedding size, hidden units, dropout, AutoML, and evaluation split.

For GPU execution in F05/F06, use `F56_GPU=true`, for example `make script5 VARIANT=v501 F56_GPU=true` or `make script6 VARIANT=v601 F56_GPU=true`.

### F06: Quantize and package for edge

```bash
make variant6 VARIANT=v601 PARENT=v501 DEPLOY_TARGET=esp32 REQUIRE_INT8=true
make script6 VARIANT=v601
make check6 VARIANT=v601
make register6 VARIANT=v601
```

F06 uses Docker for reproducible packaging in the default flow.

Set `F56_GPU=true` to switch F06 to the GPU image and pass `--gpus all` to `docker run`.

### F07: Validate a model on edge hardware

```bash
# make variant7 VARIANT=v701 PARENT=v601 PLATFORM=esp32 MTI_MS=100000
make variant7 VARIANT=v701 PARENT=v601 PLATFORM=esp32 MTI_MS=100 TIME_SCALE=0.01 VIRTUAL=true
make script7 VARIANT=v701
make check7 VARIANT=v701
make register7 VARIANT=v701
```

When the variant has `parameters.virtual: true`, `make script7` automatically delegates the ESP32 virtual run to the Docker runner (`scripts/esp32_virtual/Dockerfile`). This keeps the same command on Windows, Linux, and macOS: the host only needs Docker, while ESP-IDF, QEMU, `socat`, and the Python dependencies live inside the container.

You can also run F07 step by step:

```bash
make script7-prepare-build VARIANT=v701
make script7-flash-run VARIANT=v701
make script7-post VARIANT=v701
```

#### ESP32 flash size and large firmware images

By default, F07 keeps the ESP-IDF template configuration unchanged. For ESP32 this normally means the default 2 MB flash setting and the default single-app partition table. This is intentional: it reflects the conservative baseline and avoids silently assuming a larger physical board.

Some edge builds can produce a firmware image that is larger than the default app partition, especially when TensorFlow Lite Micro, several operators, tracing code, and an embedded model are linked into the final binary. In that case `idf.py build` may fail with an error similar to:

```text
app partition is too small for binary MLOps4OFP.bin
```

For a board or QEMU target with a larger flash, declare the intended flash size when creating the variant:

```bash
make variant7 VARIANT=v701 PARENT=v601 PLATFORM=esp32 MTI_MS=100 TIME_SCALE=0.01 ESP_FLASH_MB=4
```

This stores the value in `params.yaml`:

```yaml
parameters:
  esp_flash_size_mb: 4
```

During `script7-prepare-build`, F07 then generates a variant-local `partitions.csv` and ESP-IDF defaults inside:

```text
executions/f07_modval/<variant>/esp32_project/
```

The repository template itself remains unchanged. If the firmware does not fit in the configured partition, F07 records the condition as a deployment incompatibility, for example:

```yaml
edge_capable: false
reason: firmware_too_large_for_partition
```

Use `ESP_FLASH_MB=4` only when the physical ESP32 board, or the QEMU simulation target, is meant to provide 4 MB of flash. If the physical board has only 2 MB, a firmware that requires a larger app partition is not deployable to that board without reducing the firmware/model footprint or changing hardware.

### F08: Validate a multi-model edge system

```bash
make variant8 VARIANT=v801 PARENTS=v702,v703 PLATFORM=esp32 MTI_MS=100
make script8 VARIANT=v801
make check8 VARIANT=v801
make register8 VARIANT=v801
```

F08 also supports manual and ILP-based selection modes.

## Where Outputs Go

During execution, the pipeline writes generated files to your local workspace under executions/.

Examples of generated content:

1. params.yaml and outputs.yaml for each variant.
2. Reports, catalogs, metrics, and calibration datasets.
3. Model binaries and quantized artifacts.
4. Edge build outputs and runtime logs for hardware phases.

These files are local project outputs and are intentionally not versioned in this public repository.

## DVC, MLflow, and Git Responsibilities

The intended split is:

1. Git in this public repository stores the reusable pipeline code only.
2. DVC stores large binary artifacts generated by your project.
3. MLflow stores experiment tracking data for your project.
4. If your project uses its own Git repository, that repository is configured through your setup file.

## Troubleshooting

### Setup validation fails

Run:

```bash
make check-setup
```

Then inspect your setup file, Python version, DVC backend, and MLflow endpoint.

### A phase fails after a parent changes

The pipeline tracks parent-child relationships across variants. Re-run the affected phase after fixing the parent or create a new variant that references the updated parent.

### Edge execution fails on serial or flash steps

Check:

1. That the target board is connected.
2. That the serial port is correct.
3. That your user has permission to access the port.
4. That Docker and board toolchains are installed if the selected phase requires them.

## Additional References

1. [DEVELOPERS.md](DEVELOPERS.md) for contributors and maintainers.
2. [setup/local.yaml](setup/local.yaml) and [setup/remote.yaml](setup/remote.yaml) as setup templates.