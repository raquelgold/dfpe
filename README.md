# DFPE: Floor Plans from 360° Video

Estimates the **floor plan** of the indoor parts of a **360° video**. It cuts the
video into indoor clips, estimates the room layout in each frame (HorizonNet
or LGT-Net), tracks the camera with visual SLAM (stella_vslam), and combines
both into a floor plan with direct_360_FPE.

Where the video goes indoors is decided by the companion project,
[raquelgold/indoor_outdoor_detection](https://github.com/raquelgold/indoor_outdoor_detection).
Its output, `transitions_confirmed.csv`, is this project's input.

## Demo

![Floor plan built while walking through a building (video 097)](docs/demo_097.gif)

Video 097. Left: the original video with the INDOOR/OUTDOOR state. Right:
while indoors, the walls found so far (cyan) and the camera path; once
outside, the finished floor plan. Made with
`dpfe_pipeline/visualize_video_map_animated.py` (see [Visualisations](#visualisations)).

---

## `direct_360_FPE` provenance

A copy of [EnriqueSolarte/direct_360_FPE](https://github.com/EnriqueSolarte/direct_360_FPE)
at commit `6a41911`, with local changes to 16 files (scale recovery, room-shape
solver, the `*_coords.json` export in `utils/eval_utils.py`, config). The full
diff against upstream is in [direct_360_FPE/LOCAL_CHANGES.patch](direct_360_FPE/LOCAL_CHANGES.patch).
Upstream's `mp3d_fpe_dataset/` helper folder was removed locally (the dataset
itself is on [HuggingFace](https://huggingface.co/datasets/EnriqueSolarte/mp3d_fpe)).

---

## Setup

### External dependencies

These are not in the repo. Default locations are under `$HOME`; each can be
overridden with the environment variable shown (read by `dpfe_paths.py`).

| Dependency | Default location | Env var | Source |
|---|---|---|---|
| HorizonNet | `~/HorizonNet` | `HORIZONNET_DIR` | [sunset1995/HorizonNet](https://github.com/sunset1995/HorizonNet) @ `c9a7df9` |
| LGT-Net | `~/LGT-Net` | `LGTNET_DIR` | [zhigangjiang/LGT-Net](https://github.com/zhigangjiang/LGT-Net) @ `0045359` |
| stella_vslam (library) | installed to `/usr/local/lib` | – | [stella-cv/stella_vslam](https://github.com/stella-cv/stella_vslam) @ `8ac1be4` |
| stella_vslam_examples `run_image_slam` | `~/stella_vslam_examples/build/run_image_slam` | `STELLA_SLAM_BIN` | [stella-cv/stella_vslam_examples](https://github.com/stella-cv/stella_vslam_examples) @ `defc69e` |
| ORB vocabulary | `~/orb_vocab.fbow` | `ORB_VOCAB` | [stella-cv/FBoW_orb_vocab](https://github.com/stella-cv/FBoW_orb_vocab) |
| venv interpreter | `~/venv/bin/python` | `DFPE_VENV_PY` | see below |

### Model checkpoints

| Model | Path |
|---|---|
| HorizonNet (mp3d) | `~/HorizonNet/ckpt/resnet50_rnn__mp3d.pth` |
| LGT-Net (mp3d) | `~/LGT-Net/checkpoints/SWG_Transformer_LGT_Net/mp3d/best.pkl` |

### Python environments

| Env | Used by | Key packages |
|---|---|---|
| `~/venv` (Python 3.12) | all `dpfe_pipeline/` glue scripts | `opencv-python`, `pandas`, `numpy`, `scipy`, `matplotlib`, `pillow`, `tqdm` |
| conda `dfpe` | `run_horizonnet.py`, `run_dfpe.py` → `direct_360_FPE/main_eval_scene.py` | `torch` 2.4.1+cu118, HorizonNet deps, `direct_360_FPE/requirements.txt` |
| conda `lgtnet` | `run_lgtnet.py` (imports LGT-Net source) | `torch==1.7.1`, `LGT-Net/requirements.txt` |

```bash
# dfpe (HorizonNet + direct_360_FPE)
conda create -n dfpe python=3.9 -y && conda activate dfpe
pip install torch --index-url https://download.pytorch.org/whl/cu118
pip install -r direct_360_FPE/requirements.txt
pip install -e direct_360_FPE

# lgtnet
conda create -n lgtnet python=3.8 -y && conda activate lgtnet
pip install -r ~/LGT-Net/requirements.txt
```

### SLAM program

Build and install stella_vslam first (see its README), then:

```bash
cd ~/stella_vslam_examples
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j"$(nproc)"
```

**The SLAM camera config's `cols`/`rows` must equal the video's pixel size
exactly**, because clips are written at the video's native resolution.
`run_slam.py` picks a config from `configs/stella_vslam/` by resolution and fps.

For a new resolution/fps, copy `config_fps30_5760.yaml`, set `Camera.cols`,
`Camera.rows` and `Camera.fps` (check with `ffprobe`), and pass it with
`run_slam.py --config`. The video **must be equirectangular (360°)**.

---

## Running it

All commands run from the repo root.

**Input:** `data/videos/{ID}/transitions_confirmed.csv`, made by the
[Indoor/Outdoor detection](https://github.com/raquelgold/indoor_outdoor_detection)
project. The 23 videos 081–107 are already included.

```bash
VIDEO=~/lock_direction/new_lock_direction/VID_20260603_161558_00_097.mp4

# HorizonNet (default)
~/venv/bin/python dpfe_pipeline/pipeline.py --video "$VIDEO"
# LGT-Net
~/venv/bin/python dpfe_pipeline/pipeline.py --video "$VIDEO" --layout_model lgtnet \
    --results_root outputs/dfpe_results_lgt
```

`--video_dir` defaults to `data/videos/{ID}` (the number at the end of the file
name). Use a separate `--results_root` per layout model: both share the same
clips, so their outputs would otherwise land in the same `{id}_{n}` folders.
`--fps` must match the video (default 30).

| Stage | horizonnet | lgtnet |
|---|---|---|
| 1 | extract_indoor_clips | extract_indoor_clips |
| 2 | HorizonNet (all frames) | SLAM |
| 3 | SLAM | setup_scene |
| 4 | setup_scene | LGT-Net (keyframes only) |
| 5 | DFPE | DFPE |
| 6 | generate_maps | generate_maps |

`--from_stage N` / `--to_stage N` run a sub-range. Results land in
`outputs/dfpe_results/{ID}_{clip_index}/`, one folder per indoor clip.

### Visualisations

```bash
# Top-down GIF of the map being built
~/venv/bin/python dpfe_pipeline/generate_map_animation.py --results_dir outputs/dfpe_results/097_0

# Extra "unified" floor plan (all walls, no room separation)
~/venv/bin/python dpfe_pipeline/generate_maps.py --results_dir outputs/dfpe_results/097_0 --unified

# Side-by-side video (the demo above): original + indoor/outdoor | map being built / final map
~/venv/bin/python dpfe_pipeline/visualize_video_map_animated.py \
    --video "$VIDEO" --video_dir data/videos/097 \
    --output outputs/097_indoor_map_viz.mp4
```

---

## Experiments

Not part of the pipeline. Kept for reference; they depend on run data that is
not in the repo.

- `experiments/slam_scale_study/`: SLAM trajectory vs. MP3D-FPE ground truth
  on scene `1LXtFkjw3qL` (scale ratios, run-to-run comparisons). Reads
  `direct_360_FPE/slam_output/` and `direct_360_FPE/mp3d_fpe_dataset/`.
- `experiments/room_overlap_clipping/clip_room_overlaps.py`: removes overlaps
  between DFPE room polygons with Shapely (tried on video033/035 with LGT-Net).

---

## Original repositories and resources

### Used by the pipeline

| Repository | What it is used for here |
|---|---|
| [EnriqueSolarte/direct_360_FPE](https://github.com/EnriqueSolarte/direct_360_FPE) | Floor-plan estimation (DFPE); a modified copy is in `direct_360_FPE/` |
| [sunset1995/HorizonNet](https://github.com/sunset1995/HorizonNet) | Room-layout model (default) |
| [zhigangjiang/LGT-Net](https://github.com/zhigangjiang/LGT-Net) | Room-layout model (alternative) |
| [stella-cv/stella_vslam](https://github.com/stella-cv/stella_vslam) | Visual SLAM library (camera trajectory) |
| [stella-cv/stella_vslam_examples](https://github.com/stella-cv/stella_vslam_examples) | `run_image_slam`, the SLAM program the pipeline calls |
| [stella-cv/FBoW_orb_vocab](https://github.com/stella-cv/FBoW_orb_vocab) | ORB vocabulary file for SLAM |
| [raquelgold/indoor_outdoor_detection](https://github.com/raquelgold/indoor_outdoor_detection) | Companion project: produces `transitions_confirmed.csv` |

### Datasets

| Resource | What it is used for here |
|---|---|
| [MP3D-FPE (HuggingFace)](https://huggingface.co/datasets/EnriqueSolarte/mp3d_fpe) | Reference scene `1LXtFkjw3qL` for the SLAM/scale experiments in `experiments/slam_scale_study/` |

### Explored during the project (not used by the final pipeline)

| Repository | |
|---|---|
| [EnriqueSolarte/robust_360_8PA](https://github.com/EnriqueSolarte/robust_360_8PA) | Camera-pose estimation for 360° images |
| [DepthAnything/Video-Depth-Anything](https://github.com/DepthAnything/Video-Depth-Anything) | Video depth estimation |
| [mvlchallenge/mvl_toolkit](https://github.com/mvlchallenge/mvl_toolkit) | Multi-view layout toolkit |
| [EnriqueSolarte/ray_casting_mlc](https://github.com/EnriqueSolarte/ray_casting_mlc) | Multi-view layout consistency (self-training) |
