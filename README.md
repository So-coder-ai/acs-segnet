# ACS-SegNet H&E Segmentation Website

FastAPI website for uploading H&E stained images and generating ACS-SegNet binary masks, probability heatmaps, and overlays.

## Render Deployment

This project is configured for Render with:

- `render.yaml` Blueprint configuration.
- `build.sh` to install dependencies, clone ACS-SegNet if needed, and patch SegFormer initialization.
- `.python-version` pinned to Python `3.11.9`.
- CPU-only PyTorch in `requirements.txt`.
- One checkpoint by default to reduce memory pressure.

## Checkpoint

Default deployed checkpoint:

```text
checkpoints/ACSSegNet_fold1_best.pth
```

The app supports multiple checkpoints through `MODEL_PATHS`, but a 3-fold ensemble is heavy on CPU hosting. Use one fold unless you upgrade the Render instance.

## Required Render Environment Variables

These are already included in `render.yaml`, but you can also set/override them manually in Render:

```text
PYTHON_VERSION=3.11.9
MODEL_PATHS=checkpoints/ACSSegNet_fold1_best.pth
CACHE_MODELS=0
TORCH_NUM_THREADS=1
IMG_SIZE=256
MASK_THRESHOLD=0.5
HF_HUB_DISABLE_SYMLINKS_WARNING=1
```

## Run Locally

PowerShell:

```powershell
cd C:\Users\narai\Documents\Codex\2026-07-08\build\outputs\acs-segnet
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
.\.venv\Scripts\Activate.ps1
$env:MODEL_PATHS="checkpoints\ACSSegNet_fold1_best.pth"
$env:CACHE_MODELS="0"
$env:TORCH_NUM_THREADS="1"
python -m uvicorn app:app --reload
```

Open `http://127.0.0.1:8000`.

## Deploy on Render

1. Push this folder to GitHub.
2. Keep checkpoint files in Git LFS.
3. In Render, choose `New -> Blueprint`.
4. Select the GitHub repo.
5. Render reads `render.yaml` automatically.
6. After deployment, visit `/health`.

Expected health fields:

```json
{
  "ok": true,
  "checkpoint_exists": true,
  "acs_repo_exists": true,
  "model_error": null
}
```

## Notes

- First prediction is slow because the model loads lazily.
- The app uses CPU mode by default.
- If prediction crashes on Render, upgrade RAM or keep `MODEL_PATHS` to a single fold.

## Research Notebooks

This repository also contains the research notebooks used to reproduce and adapt ACS-SegNet
for the SegPath dataset, as described in our accompanying paper (link once posted to arXiv).

- **`ACS_SegNet_SegPath_CORRECTED.ipynb`** — the primary notebook. Contains the 3-fold
  cross-validation training run, the mask-encoding bug fix, the corrected polynomial LR
  schedule, and both evaluation protocols (repository per-batch metrics and the paper's
  micro-averaging protocol). This is the notebook that produced all results tables in the paper.
- **`ACS_SegNet_Epithelium_only.ipynb`** — single-tissue-class training run (epithelium only).
- **`ACS_SegNet_SmoothMuscle_only.ipynb`** — single-tissue-class training run (smooth muscle only).
- **`AACS_SegNet_Endothelium_only.ipynb`** - single-tissue-class training run (endothelium muscle only).
These notebooks clone the official ACS-SegNet implementation from
[Torbati et al.](https://github.com/NimaTorbati/ACS-SegNet) at runtime and modify only the
data pipeline, training loop, and evaluation code — the model architecture itself
(`model.py` / `DualEncoderUNet`) is unmodified from the original repository.

### Model Weights

We do not include a standalone `train.py`, since training was run interactively in the
notebooks above. The trained checkpoint used for both evaluation and the deployed web app
(`checkpoints/ACSSegNet_fold1_best.pth`) is the Fold 1 best-validation-IoU checkpoint produced
by `ACS_SegNet_SegPath_CORRECTED.ipynb`, tracked via Git LFS in this repository.

### Data

The SegPath dataset is **not redistributed in this repository**. Download it directly from
[dakomura.github.io/SegPath](https://dakomura.github.io/SegPath/), where it is released under
a CC-BY-NC-SA 4.0 license (non-commercial use only). The notebooks expect the data in the
`SegPath_sorted/` folder structure documented in the notebook's configuration cell.