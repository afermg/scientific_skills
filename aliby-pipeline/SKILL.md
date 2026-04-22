---
name: aliby-pipeline
description: >
  Run the aliby image analysis pipeline with nahual cellpose/trackastra GPU
  servers. TRIGGER when: user asks to run extraction, segmentation, profiling,
  or any of the timelapse/cellpainting/viability/cycle pipelines, mentions
  nahual servers, or wants to start/stop/monitor pipeline runs. Also trigger
  on "run nb01", "run pipelines", "start cellpose servers", "start trackastra".
---

# Aliby Pipeline Runner

Run the aliby image analysis pipeline (`notebooks/nb01_extract_profiles.py`)
with nahual cellpose and trackastra GPU servers for segmentation and tracking.

## Architecture overview

Three components:

1. **Nahual cellpose servers** -- standalone GPU processes doing cell/nuclei
   segmentation via IPC sockets (`ipc:///tmp/cellpose{N}.ipc`)
2. **Nahual trackastra server** -- standalone GPU process for cell tracking
   via IPC socket (`ipc:///tmp/trackastra.ipc`)
3. **Pipeline workers** -- joblib parallel workers that read images, send
   them to nahual servers for segmentation/tracking, then extract
   morphological features with cp_measure

Pipeline entry point: `notebooks/nb01_extract_profiles.py` (marimo notebook).
The `run_pipelines()` function is importable for scripted runs.

## Notebook layout

| Notebook | Purpose |
|----------|---------|
| nb01_extract_profiles | Segmentation + feature extraction (this skill) |
| nb02_segmentation_audit | Review segmentation quality |
| nb03_tracking_quality | Review trackastra tracks |
| nb04_aggregate_profiles | Aggregate single-cell → well/site |
| nb05_phenotypic_scoring | Cross-batch phenotypic scoring |
| nb06_umap_explorer | UMAP visualisation |
| nb07_normalization | Per-batch normalization |
| nb09_cross_batch_normalization | Cross-batch normalization |
| nb10_feature_importance | Feature importance analysis |
| nb11_assay_concordance | Inter-assay concordance |

## Critical: CUDA stub library issue

Nix environments include a stub `libcuda.so` from `cuda_cudart-*-stubs` that
shadows the real NVIDIA driver at `/run/opengl-driver/lib/libcuda.so`. This
causes `Error 34: CUDA driver is a stub library` and forces CPU fallback.

**Fix**: prepend `/run/opengl-driver/lib` to `LD_LIBRARY_PATH`:

```bash
export LD_LIBRARY_PATH=/run/opengl-driver/lib:$LD_LIBRARY_PATH
```

Already fixed in this project's `flake.nix`. Pass it explicitly when
launching nahual servers via `nix run`.

## Starting nahual servers with `nix run`

Both `nahual_models/cellpose` and `nahual_models/trackastra` flakes expose
`apps.default` that runs `server.py` with a socket argument. Prefer
`nix run github:afermg/...` so there are no local checkouts to maintain.

### Cellpose servers

Each server loads CellposeSAM (~4 GB VRAM). On 2× RTX A6000 (49 GB each),
run up to 6 servers per GPU (12 total). Launch each in a `screen` session:

```bash
# 6 servers on GPU 0
for i in $(seq 0 5); do
  screen -dmS cellpose$i bash -c \
    "export LD_LIBRARY_PATH=/run/opengl-driver/lib:\$LD_LIBRARY_PATH && \
     export CUDA_VISIBLE_DEVICES=0 && \
     nix run github:afermg/nahual_cellpose -- ipc:///tmp/cellpose$i.ipc"
done

# 6 servers on GPU 1
for i in $(seq 6 11); do
  screen -dmS cellpose$i bash -c \
    "export LD_LIBRARY_PATH=/run/opengl-driver/lib:\$LD_LIBRARY_PATH && \
     export CUDA_VISIBLE_DEVICES=1 && \
     nix run github:afermg/nahual_cellpose -- ipc:///tmp/cellpose$i.ipc"
done
```

### Trackastra server

A single instance is enough (tracking is fast):

```bash
screen -dmS trackastra bash -c \
  "export LD_LIBRARY_PATH=/run/opengl-driver/lib:\$LD_LIBRARY_PATH && \
   export CUDA_VISIBLE_DEVICES=1 && \
   nix run github:afermg/nahual_trackastra -- ipc:///tmp/trackastra.ipc"
```

Replace flake URLs with whatever is canonical for your account/org. If the
exact URL is unknown, fall back to `nix run /path/to/nahual_models/<name>`
against a local checkout.

### Verifying servers

Wait ~60-90 seconds for build + model loading, then test:

```python
from nahual.process import dispatch_setup_process
import numpy as np

setup, process = dispatch_setup_process('cellpose')
info = setup({}, address='ipc:///tmp/cellpose0.ipc')
print('Setup:', info)  # expect device: cuda:0

img = np.random.randint(0, 255, (1, 128, 128), dtype=np.uint16)
result = process(img, address='ipc:///tmp/cellpose0.ipc')
print('Result shape:', result.shape)  # (128, 128)
```

If setup returns `{}`, the model failed to load -- check the screen session
for errors (usually CUDA OOM from too many servers on one GPU).

## Running the pipeline

```python
from pathlib import Path
from notebooks.nb01_extract_profiles import run_pipelines

batches_and_assays = [
    ('<batch_id>', '<assay>'),
    ...
]
input_dir = '<path to batches root>'
profiles_path = Path('<path to profiles output root>')
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
- **ncores**: parallel site workers (joblib). ~18 on a 192-core machine.
- **extract_ncores**: cores for cp_measure feature extraction per site.
- **nahual_addresses**: list of IPC addresses; assigned round-robin to sites.
- **overwrite**: `False` skips already-processed sites.

Available `(batch, assay)` pairs come from
`notebooks/nb01_extract_profiles.get_batches_assays()` -- call it to discover
the valid list for the current dataset instead of hard-coding.

Timelapse assays include trackastra tracking as a global step (uses the
trackastra server); non-timelapse assays don't need it.

### Without nahual (local cellpose)

Pass `nahual_addresses=None` to use local CellposeSAM. The model is loaded
in-process once per worker, so reduce `ncores` to avoid GPU OOM. Requires
`LD_LIBRARY_PATH=/run/opengl-driver/lib:$LD_LIBRARY_PATH` in the environment.

## Resource guidelines (192-core, 754 GB RAM, 2× RTX A6000)

| Config | CPU | RAM | Notes |
|--------|-----|-----|-------|
| ncores=18, 12 cellpose servers | ~35-60% | ~270-400 GB | Recommended |
| ncores=24, no nahual (local GPU) | ~50-90% | 400+ GB | OOM risk |
| ncores=12, no nahual (local GPU) | ~35-55% | ~270 GB | Conservative |

**RAM warning threshold**: >500 GB used (~66%). With no swap, the OOM killer
will terminate workers silently (no traceback -- just `resource_tracker`
leaked-object warnings at the end of the log).

## Monitoring

```bash
# Resource usage
top -bn1 | head -5 && free -h && \
  nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total --format=csv

# Pipeline progress
grep -c "Saving"    <log>   # completed steps
grep -c "Skipping"  <log>   # already-done sites
grep -c "Exception" <log>   # failures

# OOM-killer signature (no traceback, just at end):
#   resource_tracker: There appear to be N leaked semlock/folder objects
```

## Stopping servers

```bash
# Kill all screen sessions for nahual servers
screen -ls | grep -E "cellpose|trackastra" | awk -F. '{print $1}' | \
  xargs -I{} screen -X -S {} quit

# Kill any orphaned server processes
pkill -f "python server.py ipc://"

# Clear leaked GPU memory (verify first with nvidia-smi)
nvidia-smi --query-compute-apps=pid --format=csv,noheader | xargs kill
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `unpack requires a buffer of 2 bytes` | Server returned `{}` instead of numpy | Server crashed processing -- check screen for CUDA OOM |
| `ConnectionRefused` on IPC sockets | Server not up yet | Wait 60-90s after `nix run` for build + model load |
| `CUDA driver is a stub library` | Wrong `libcuda.so` found first | Prepend `/run/opengl-driver/lib` to `LD_LIBRARY_PATH` |
| Pipeline hangs silently | All workers blocked on dead server | Kill pipeline, restart servers, rerun with `overwrite=False` |
| Silent death, leaked-semlock warnings | OOM killer | Reduce `ncores` or number of cellpose servers |
| `ModuleNotFoundError: analysis` | Old path; notebooks moved to `notebooks/` | Import from `notebooks.nb01_extract_profiles` |
