# GreenCrossingAI: Notebook Quick Start

This repo holds notebooks that run MegaDetector over camera-trap footage (video and still images) to detect animals and produce annotated output for review.

> Want to contribute changes back to this repo? See [CONTRIBUTING.md](CONTRIBUTING.md) for the team's conventions first.

## Repo Structure

```
Notebooks/
├── linux/          # notebooks for Linux / JupyterHub environments
├── windows/        # notebooks for Windows machines with an NVIDIA GPU (a.k.a. "Tarazed")
└── old-Notebooks/  # deprecated predecessors, kept for reference only (not maintained)
```

Each platform has two notebooks: one for video, one for still images.

| Platform | Video | Images |
|---|---|---|
| Windows / Tarazed | `Notebooks/windows/Video_Processing_Windows_2026.ipynb` | `Notebooks/windows/BatchProcessing_with_filedialog.ipynb` |
| Linux / JupyterHub | `Notebooks/linux/Batch-Video_Processing_2026.ipynb` | `Notebooks/linux/BatchProcessing_with_filedialog.ipynb`¹ |

¹ Currently a straight copy of the Windows images notebook and not yet adapted for headless use. It still relies on tkinter file-picker dialogs, which need a display. Use the video notebook on Linux for now if you need headless/no-display operation.

## Setup: Windows / Tarazed (NVIDIA GPU)

**Platform:** Windows with an NVIDIA GPU. Match CUDA/cuDNN to the ML package you install.
**Python:** 3.9+ recommended.

1. **Create and activate** a conda/venv environment.

2. **Install packages:**

```bash
   pip install jupyterlab jupyter numpy pandas opencv-python pillow tqdm matplotlib
   pip install megadetector  # pulls in a compatible torch/torchvision/YOLO stack automatically
```

3. **Register the kernel system-wide, not per-user.** On a shared machine, `python -m ipykernel install --user ...` registers the kernel under `%APPDATA%\jupyter`, which Jupyter's data-file search order checks before the machine-wide location. As a result, a per-user kernel can silently shadow the one other accounts are meant to use. Install it machine-wide instead:

```bash
   jupyter kernelspec install "<path to the environment's kernelspec>" --name megadetector
```

   (No `--user`, no `--prefix`.) This lands in the fixed `%PROGRAMDATA%\jupyter\kernels\` location, independent of which environment's Python is running, so every account on the machine sees the same kernel. If a per-user copy already exists from an earlier install, remove it; otherwise it continues to shadow the machine-wide one for any session launched from that account.

4. **Model weights:** Stage the MegaDetector model weights (e.g. `md_v5a.0.1.pt`) at a fixed, shared local path, such as `C:\ProgramData\megadetector_models\`, rather than relying on the package's auto-download. On a shared machine, the package's default cache lives under each user's own `%TEMP%`, so a different account than the one that downloaded the weights will attempt its own re-download the first time it runs. This can surface as a `CERTIFICATE_VERIFY_FAILED` error that has nothing to do with the certificate itself. Point every notebook at the same shared file and check that it exists before any download is attempted.

5. **Run:**

```bash
   jupyter lab
```

   Open the notebook for what you're processing (see table above) and run top-to-bottom. Folder and model selection happen via file-picker dialogs.

## Setup: Linux / JupyterHub

**Platform:** Any Linux JupyterHub instance (e.g. NAIRR Jetstream2, TLJH), with GPU access recommended.
Prefer conda if available; venv works too.

1. **Install packages:**

```bash
   pip install jupyterlab jupyter numpy pandas opencv-python pillow tqdm matplotlib
   pip install megadetector
```

2. **Data & storage:** Use scratch or project persistent storage for large video/frame sets. Update `DATA_DIR` / `OUTPUT_DIR` in the notebook's top cells accordingly.

3. **Model weights:** Same advice as above. Stage weights at a fixed, persistent local path (not a temp directory) and point the notebook at that exact file, rather than an auto-downloading shorthand name.

4. **Run:** Open the video-processing notebook (see table above) in JupyterLab, edit the top cells to set paths, then run cells top-to-bottom.

## Verifying Your Install

Before assuming a bug is in the code, confirm which MegaDetector the environment is actually running:

```bash
python -c "import megadetector as m; print(m.__file__)"
python -m pip show megadetector
```

If the printed file path does not fall inside the `Location` that `pip show` reports, something earlier in the module search path is shadowing the install. Most commonly this is a machine-wide `PYTHONPATH` entry pointing at an old clone. Check it:

```bat
:: cmd
echo %PYTHONPATH%
```

```powershell
# PowerShell
$env:PYTHONPATH
```

A shadowed install upgrades without error and without visibly changing anything, so this check is worth running after any `pip install --upgrade megadetector`, not just when something looks broken.

## Common Notes

- If a `requirements.txt` exists, prefer:

```bash
  pip install -r requirements.txt
```

- Always edit the top cells in each notebook to set `DATA_DIR`, `OUTPUT_DIR`, and `MEGADETECTOR_PATH` / `MODEL_FILE` before running. They won't point at your data by default.

- For GPU usage, make sure your torch build matches your NVIDIA driver's supported CUDA version (check `nvidia-smi`). A mismatch silently falls back to CPU rather than erroring.

- Invoke MegaDetector via `subprocess.run([sys.executable, "-m", "megadetector...", ...])`, not a bare `python` or `!python`. A bare interpreter name can resolve to a different Python than the one running the notebook (e.g. an old 3.8 env still on `PATH`), which shows up as `ModuleNotFoundError: megadetector` even though the package is installed correctly in the kernel's own environment.

- The video batch loop captures subprocess output and skips a video that fails rather than halting the run; failed videos are logged by filename rather than silently dropped. If you're extending the batch loop, preserve this behavior. A `check=True` call with stderr uncaptured is what previously caused a single bad video to take down an entire run.