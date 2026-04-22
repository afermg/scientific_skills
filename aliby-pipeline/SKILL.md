---
name: aliby-pipeline
description: >
  Run the aliby image analysis pipeline with nahual cellpose/trackastra GPU
  servers. TRIGGER when: user asks to run extraction, segmentation, profiling,
  or the timelapse/cellpainting/viability/cycle pipeline, mentions nahual
  servers, or wants to start/stop/monitor pipeline runs. Also trigger on
  "run nb01", "run pipelines", "start cellpose servers", "start trackastra".
---

# Aliby Pipeline Runner

Run the GSK/Broad image analysis pipeline (nb01_extract_profiles) with nahual
cellpose and trackastra GPU servers for segmentation and tracking.

## Architecture overview

The pipeline has three components:

1. **Nahual cellpose servers** -- standalone GPU processes that do cell/nuclei
   segmentation via IPC sockets (`ipc:///tmp/cellpose{N}.ipc`)
2. **Nahual trackastra server** -- standalone GPU process for cell tracking
   via IPC socket (`ipc:///tmp/trackastra.ipc`)
3. **Pipeline workers** -- joblib parallel workers that read images, send them
   to nahual servers for segmentation/tracking, then extract morphological
   features

The servers live in separate nix flakes at `/home/amunoz/projects/nahual_models/`.

## Critical: CUDA stub library issue

Nix environments include a stub `libcuda.so` from `cuda_cudart-*-stubs` that
shadows the real NVIDIA driver at `/run/opengl-driver/lib/libcuda.so`. This
causes `Error 34: CUDA driver is a stub library` and makes PyTorch fall back
to CPU.

**Fix**: Always prepend `/run/opengl-driver/lib` to `LD_LIBRARY_PATH`:

```bash
export LD_LIBRARY_PATH=/run/opengl-driver/lib:$LD_LIBRARY_PATH
```

This is already fixed in this project's `flake.nix` but the nahual model
flakes (`cellpose/`, `trackastra/`) still have the bug. Pass the env var
explicitly when launching their servers.

## Starting nahual servers

### Cellpose servers

Each server loads CellposeSAM (~4 GB VRAM). With 2x RTX A6000 (49 GB each),
run up to 6 servers per GPU (12 total). Use `screen` + `nix develop`:

```bash
# 6 servers on GPU 0
for i in $(seq 0 5); do
  screen -dmS cellpose$i bash -c "cd /home/amunoz/projects/nahual_models/cellpose && \
    nix develop . --command bash -c \
    'export LD_LIBRARY_PATH=/run/opengl-driver/lib:\$LD_LIBRARY_PATH && \
     export CUDA_VISIBLE_DEVICES=0 && \
     python server.py ipc:///tmp/cellpose$i.ipc'"
done

# 6 servers on GPU 1
for i in $(seq 6 11); do
  screen -dmS cellpose$i bash -c "cd /home/amunoz/projects/nahual_models/cellpose && \
    nix develop . --command bash -c \
    'export LD_LIBRARY_PATH=/run/opengl-driver/lib:\$LD_LIBRARY_PATH && \
     export CUDA_VISIBLE_DEVICES=1 && \
     python server.py ipc:///tmp/cellpose$i.ipc'"
done
```

### Trackastra server

Single instance is sufficient (tracking is fast):

```bash
screen -dmS trackastra bash -c "cd /home/amunoz/projects/nahual_models/trackastra && \
  nix develop . --command bash -c \
  'export LD_LIBRARY_PATH=/run/opengl-driver/lib:\$LD_LIBRARY_PATH && \
   export CUDA_VISIBLE_DEVICES=1 && \
   python server.py ipc:///tmp/trackastra.ipc'"
```

### Verifying servers

Wait ~60-90 seconds for nix develop + model loading, then test:

```python
from nahual.process import dispatch_setup_process
import numpy as np

setup, process = dispatch_setup_process('cellpose')
info = setup({}, address='ipc:///tmp/cellpose0.ipc')
print('Setup:', info)  # Should show device: cuda:0

img = np.random.randint(0, 255, (1, 128, 128), dtype=np.uint16)
result = process(img, address='ipc:///tmp/cellpose0.ipc')
print('Result shape:', result.shape)  # Should be (128, 128)
```

If setup returns `{}` (empty dict), the model failed to load -- check the
screen session for errors (usually CUDA OOM from too many servers on one GPU).

## Running the pipeline

```python
from pathlib import Path
from analysis.nb01_extract_profiles import run_pipelines

batches_and_assays = [
    ('ELN201687', 'timelapse_vs'),
    ('ELN374825', 'timelapse'),
]
input_dir = '/datastore/alan/gsk/batches'
profiles_path = Path('/datastore/alan/gsk/aliby_output/standard_output')
nahual_addresses = [f'ipc:///tmp/cellpose{i}.ipc' for i in range(12)]

run_pipelines(
    batches_and_assays=batches_and_assays,
    input_dir=input_dir,
    profiles_path=profiles_path,
    extract_ncores=192,
    ntps=None,
    overwrite=False,
    ncores=18,           # parallel site workers
    max_sites=None,
    nahual_addresses=nahual_addresses,
)
```

Key parameters:
- **ncores**: parallel site workers (joblib). 18 is good for 192-core machine
- **extract_ncores**: cores for feature extraction within each site (192)
- **nahual_addresses**: list of IPC addresses; assigned round-robin to sites
- **overwrite**: False skips already-processed sites

### Without nahual (local cellpose)

Pass `nahual_addresses=None` to use local CellposeSAM. This loads the model
in-process (one per worker), so reduce ncores to avoid GPU OOM. Requires
`LD_LIBRARY_PATH=/run/opengl-driver/lib:$LD_LIBRARY_PATH` in the environment.

## Available batches and assays

| Batch | Assay | Type |
|-------|-------|------|
| ELN201687 | timelapse_vs | Timelapse with viability stain (2 channels, trackastra) |
| ELN201687 | cycle | Cell cycle |
| ELN201687 | viability | Viability |
| ELN201687 | cellpainting | Cell Painting (5 channels) |
| ELN374825 | timelapse | Timelapse (1 channel, trackastra) |
| ELN374825 | cycle | Cell cycle |
| ELN374825 | viability | Viability |
| ELN374825 | cellpainting | Cell Painting |

Timelapse assays include trackastra tracking as a global step.

## Resource guidelines (192-core, 754 GB RAM, 2x RTX A6000)

| Config | CPU% | RAM | Notes |
|--------|------|-----|-------|
| ncores=18, 12 cellpose servers | ~35-60% | ~270-400 GB | Recommended |
| ncores=24, no nahual (local GPU) | ~50-90% | 400+ GB | Risk of OOM |
| ncores=12, no nahual (local GPU) | ~35-55% | ~270 GB | Conservative |

**RAM warning threshold**: >500 GB used (~66%). If exceeded with no swap, the
OOM killer will terminate workers silently (no traceback, just
`resource_tracker` leaked object warnings in the log).

## Monitoring

```bash
# Resource usage
top -bn1 | head -5 && free -h && nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total --format=csv

# Pipeline progress
grep -c "Saving" run_timelapse.log    # completed steps
grep -c "Skipping" run_timelapse.log  # already-done sites
grep -c "Exception" run_timelapse.log # failures

# Check for the OOM pattern (no traceback, just this at the end):
# resource_tracker: There appear to be N leaked semlock/folder objects
```

## Stopping servers

```bash
# Kill all screen sessions
screen -ls | grep -E "cellpose|trackastra" | awk -F. '{print $1}' | \
  xargs -I{} screen -X -S {} quit

# Kill any orphaned processes
pkill -f "python server.py ipc://"

# Clear leaked GPU memory (check first with nvidia-smi)
nvidia-smi --query-compute-apps=pid --format=csv,noheader | xargs kill
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `unpack requires a buffer of 2 bytes` | Cellpose server returning `{}` instead of numpy | Server failed to process -- check screen for CUDA OOM, reload model |
| `ConnectionRefused` on IPC sockets | Server not running or not ready | Start servers, wait 60-90s for model loading |
| `CUDA driver is a stub library` | Wrong `libcuda.so` in `LD_LIBRARY_PATH` | Prepend `/run/opengl-driver/lib` |
| Pipeline hangs, no log output | All workers blocked on failed servers | Kill pipeline, restart servers, rerun with `overwrite=False` |
| Silent death, leaked semlock warnings | OOM killer | Reduce ncores or number of cellpose servers |
