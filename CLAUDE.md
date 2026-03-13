# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

sunnypilot is a fork of [comma.ai's openpilot](https://github.com/commaai/openpilot), an open source driver assistance system for 300+ car makes/models. sunnypilot extends openpilot with additional features like MADS (Modular Assistive Driving System), sunnylink integration, custom neural network models, map data (mapd), and UI enhancements.

## Build & Development Commands

### Setup
```bash
tools/op.sh setup           # Install all dependencies, submodules, git-lfs
source .venv/bin/activate   # Activate Python venv (or: tools/op.sh venv)
```

### Build (C++/Cython via SCons)
```bash
scons -j$(nproc)            # Full build
scons -j4                   # Build with 4 cores
scons selfdrive/pandad/     # Build a specific target
```

### Lint
```bash
scripts/lint/lint.sh        # Full lint (ruff, ty, codespell, shebang checks)
scripts/lint/lint.sh -f     # Fast lint (skips ty and codespell)
ruff check .                # Python lint only
ruff check . --fix          # Auto-fix lint issues
```

### Tests
```bash
pytest                                          # Run all tests (auto-parallelized with -n auto)
pytest selfdrive/car/tests/test_car_interfaces.py  # Run a specific test file
pytest -k "test_function_name"                  # Run a specific test
pytest -m "not slow"                            # Skip slow tests
pytest -x                                       # Stop on first failure
pytest -n0 -s                                   # Single-process with stdout (for debugging)
```

Tests use `OpenpilotPrefix` fixtures that create isolated environments. Tests marked `tici` only run on comma device hardware. Test paths: `common/`, `selfdrive/`, `system/`, `tools/`, `cereal/`, `sunnypilot/`.

### Docker
```bash
docker build -t sunnypilot -f Dockerfile.openpilot .
```

## Architecture

### Core Layer (from openpilot)
- **`selfdrive/controls/`** — Vehicle control: `controlsd.py` (lateral/longitudinal control loop), `plannerd.py` (path planning), `radard.py` (radar processing)
- **`selfdrive/selfdrived/`** — State machine and alert management: `selfdrived.py`, `events.py`, `alertmanager.py`
- **`selfdrive/car/`** — Car interface abstraction: `card.py` (CAN communication), `car_specific.py`, `cruise.py`
- **`selfdrive/modeld/`** — Neural network driving model inference
- **`selfdrive/locationd/`** — Localization and sensor fusion
- **`selfdrive/monitoring/`** — Driver monitoring
- **`system/manager/`** — Process lifecycle management. `process_config.py` defines all managed processes and their start conditions
- **`system/ui/`** — Python/raylib-based UI with widget system (`system/ui/lib/`, `system/ui/widgets/`)
- **`system/loggerd/`** — Data logging
- **`system/hardware/`** — Hardware abstraction (TICI/comma device vs PC)
- **`common/`** — Shared utilities: `params.py` (persistent key-value store), `swaglog.py` (logging), `realtime.py`

### sunnypilot Extension Layer
sunnypilot features are in the `sunnypilot/` directory, mirroring the `selfdrive/`/`system/` structure and extending base classes:

- **`sunnypilot/mads/`** — Modular Assistive Driving System: independent lateral control that persists through brake events
- **`sunnypilot/selfdrive/controls/`** — `controlsd_ext.py` extends the base `Controls` class
- **`sunnypilot/selfdrive/car/`** — Extended car interfaces, intelligent cruise button management
- **`sunnypilot/sunnylink/`** — Cloud connectivity service (alternative to comma connect)
- **`sunnypilot/mapd/`** — Map data integration for speed limit and curve awareness
- **`sunnypilot/modeld_v2/`** — Custom model runner configurations
- **`sunnypilot/system/ui/`** — Additional UI widgets and screens (in `system/ui/sunnypilot/`)

### Extension Pattern
sunnypilot extends openpilot using class inheritance rather than forking files. Example: `Controls(ControlsExt)` in `selfdrive/controls/controlsd.py` inherits from `sunnypilot/selfdrive/controls/controlsd_ext.py`. This minimizes merge conflicts with upstream.

### Message Bus & Serialization
- **`cereal/`** — Cap'n Proto message definitions (`log.capnp`, `car.capnp`, `custom.capnp`). sunnypilot-specific messages go in `custom.capnp`
- **`msgq_repo/`** — ZeroMQ-based pub/sub messaging between processes
- Messages are accessed via `cereal.messaging` (e.g., `messaging.sub_sock('carState')`)

### Submodules
`panda/`, `opendbc_repo/`, `msgq_repo/`, `rednose_repo/`, `tinygrad_repo/`, `teleoprtc_repo/` — separate repos with their own build systems. Symlinked names (e.g., `opendbc` -> `opendbc_repo`) exist for import compatibility.

### Import Conventions
All imports must use the `openpilot.` prefix:
```python
from openpilot.selfdrive.controls.lib.drive_helpers import clip_curvature
from openpilot.sunnypilot.mads.mads import ModularAssistiveDrivingSystem
from openpilot.common.params import Params
```
Bare imports like `from selfdrive.xxx` or `from common.xxx` are banned by ruff rules.

## Code Style

- **Python 3.12**, 2-space indentation, 160 char line length
- Use `pytest` (never `unittest`), use `time.monotonic` (never `time.time`)
- Linting: `ruff` (primary), `ty` (type checking), `codespell` (spelling)
- Use `pyray.draw_font_ex` (not `draw_text`), `Widget._handle_mouse_press/release` (not `is_mouse_button_pressed/released`), `text_measure` module (not `measure_text_ex`)
- PRs target `master` branch
