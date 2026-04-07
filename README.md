# Real-Time 3D Mesh Generation Pipeline

Converts a set of RGB images (< 200) into a clean, editable triangle mesh (OBJ/PLY) in under 5 minutes on a consumer GPU.

---

## Architecture

```
Images → Blur Filter → COLMAP SfM → Track-Length Filter → Poisson Reconstruction → OBJ/PLY
```

| Stage | Method | Notes |
|-------|--------|-------|
| Frame selection | Laplacian variance blur filter | Keeps sharpest 85% of frames |
| Feature extraction | COLMAP SIFT (CPU) | Exhaustive matching for <120 images |
| SfM | COLMAP incremental mapper | Produces metric sparse point cloud |
| Background filtering | Track-length filter | Removes points seen in few views (background) |
| Surface reconstruction | Poisson (Open3D, depth=7) | Adaptive voxel size from point cloud density |
| Cleanup | Largest component + hole fill + decimation | ≤ 50K faces output |

---

## Setup

### 1. Python environment

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Requires Python 3.9+. CUDA recommended but not required.

### 2. COLMAP

```bash
# Ubuntu/Debian
sudo apt install colmap

# macOS
brew install colmap
```

> Note: Install the CPU-only build. GPU COLMAP is not required — the pipeline uses `SiftExtraction.use_gpu 0`.

### 3. Optional: background removal

```bash
pip install rembg onnxruntime
```

Used for foreground masking in the depth fusion fallback path.

---

## Usage

```bash
python run.py --input ./images --output mesh.obj
```

### Options

```
--input           Directory of input RGB images (.jpg / .jpeg / .png)
--output          Output mesh path (.obj or .ply)
--max-faces       Max faces after decimation (default: 50000)
--work-dir        Custom working directory for intermediate files
--keep-workspace  Keep intermediate files after completion
```

---

## Capture Guidelines

Mesh quality depends heavily on input quality. For best results:

| Factor | Recommendation |
|--------|---------------|
| Distance | 30–60 cm from object |
| Coverage | Full 360° orbit at 2–3 height levels |
| Speed | Move slowly — 30–40s per full orbit |
| Lighting | Bright, diffuse (avoid harsh shadows) |
| Background | Plain/dark background preferred |
| Frame count | 60–120 frames |

Check your capture sharpness before running:
```bash
python3 -c "
import cv2, numpy as np, os, sys
imgs = [f for f in os.listdir(sys.argv[1]) if f.endswith(('.jpg','.png'))]
scores = [cv2.Laplacian(cv2.imread(f'{sys.argv[1]}/{f}', 0), cv2.CV_64F).var() for f in imgs]
print(f'Frames: {len(scores)}, Min: {min(scores):.0f}, Median: {np.median(scores):.0f}, Max: {max(scores):.0f}')
print('Target: median > 50 for good results')
" ./images
```

---

## Timing Benchmark

Measured on RTX 3060, Middlebury Temple Ring dataset (47 images, 1920×1080):

| Stage | Time |
|-------|------|
| Feature extraction / SfM | ~27s |
| Reconstruction (sparse cloud) | ~4s |
| Meshing + decimation | ~37s |
| **Total** | **~68s (1.15 min)** |

---

## Output Quality

- Triangle mesh with vertex normals
- ≤ 50K faces after quadric decimation
- Degenerate triangles removed
- Largest connected component kept
- Holes filled via trimesh

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `open3d` | Normal estimation, Poisson reconstruction, decimation, export |
| `opencv-python` | Image loading, blur detection |
| `trimesh` | Hole filling |
| `torch` | Depth model inference (fallback path) |
| `transformers` | Depth Anything V2 (fallback path) |
| `rembg` | Background removal (fallback path) |
| `colmap` (binary) | SfM — must be installed separately |

---

## Troubleshooting

**Few images registered** — check sharpness scores. If median < 20, the video has too much motion blur. Move the camera more slowly.

**Empty mesh / too few points** — COLMAP needs texture to find keypoints. Dark matte objects (black plastic, metal) are challenging. Use brighter lighting or add temporary texture markers.

**COLMAP fails entirely** — falls back to OpenCV pose estimation + depth fusion automatically.

**OOM during meshing** — reduce `--max-faces` or the point cloud will be downsampled more aggressively.
