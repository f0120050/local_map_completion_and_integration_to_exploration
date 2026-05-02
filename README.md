# Frontier-Centered Local Map Completion for Autonomous Exploration

This repository contains the code, dataset generation tools, and exploration logs associated with the paper:

> **Frontier-Centered Local Map Completion for Autonomous Exploration**  
> Selma Güzel, Erkan Uslu  
> Yildiz Technical University, Department of Computer Engineering

---

## Repository Contents

| File | Description |
|------|-------------|
| `dataset_generator.zip` | Extracts frontier-centered 32×32 patches from sequential SLAM maps to construct the SDS/BDS dataset |
| `local_map_completion.zip` | Model training and evaluation code (U-Net variants, CVAE, GAN, ViT, RePaint, LSTM, ImageGPT) |
| `integrated_code_for_map_completion.zip` | ROS integration: MapCompletionBatch service node (Python, hosts the U-Net model) |
| `m-explore_20260312.zip` | Modified `explore_lite` ROS package with completion-aware frontier scoring |
| `exploration_logs_20260315.zip` | Exploration run logs for all 8 HouseExpo environments (BASE and MC configurations, 3 runs each) |
| `env_classifier_and_figure_generator.zip` | Computes structural metrics from PGM maps and generates all system-level evaluation figures |
| `supplementary.pdf` | Supplementary material: qualitative model comparisons, UNet variant results, BDS results, and dataset split details |

---

## How to Use

### Step 1: Generate the Base Simulation Environments

The Gazebo simulation environments used in this work are generated from HouseExpo floor plans.  
Follow the instructions at:  
[houseexpo_based_custom_dataset_with_gazebo](https://github.com/f0120050/houseexpo_based_custom_dataset_with_gazebo)

### Step 2: Generate the Frontier-Centered Patch Dataset

After generating the simulation environments and collecting sequential SLAM maps via robot exploration:

```bash
unzip dataset_generator.zip
cd dataset_generator
# Follow the README inside for configuration
python generate_patches.py --map_dir <path_to_maps> --output_dir <path_to_output>
```

This produces frontier-centered 32×32 input/output patch pairs.  
The dataset is split at the **environment level** to prevent data leakage.

- **SDS**: 9,418 train / 999 validation / 160 test patches  
- **BDS**: 10,418 train / 999 validation / 160 test patches (same test set as SDS)

### Step 3: Train and Evaluate Models

```bash
unzip local_map_completion.zip
cd local_map_completion
# Install requirements (see requirements files inside the zip)
python train.py --config configs/unet.yaml
python eval.py --config_model configs/unet.yaml --config_data configs/data.yaml --ckpt outputs/unet/checkpoints/best.pth
```

Available model configurations: `unet`, `unet_attention`, `unet_4layers`, `unet_new_weights`, `lama`, `cvae`, `cgan`, `seg_gan`, `vit`, `repaint`, `lstm_local`, `imagegpt_local`.

Two evaluation scopes are supported:
- **MASKED**: metrics computed only over unknown cells (primary criterion)
- **FULL**: metrics computed over the entire patch

### Step 4: Deploy the Map Completion ROS Service

```bash
unzip integrated_code_for_map_completion.zip
cd integrated_code_for_map_completion
# Place trained checkpoint at the configured path
roslaunch map_completion batch_map_completion_unet.launch
```

This launches the `MapCompletionBatch` ROS service that receives frontier patch requests and returns completed patches using the trained U-Net model. Results are cached for 30 seconds.

### Step 5: Run Exploration with Completion-Aware Frontier Scoring

```bash
unzip m-explore_20260312.zip
cd m-explore_20260312
# Build the ROS package
catkin_make
# Launch exploration
roslaunch explore_lite explore.launch
```

Set `use_completed_map=1` to enable completion-aware frontier scoring (MC configuration).  
Set `use_completed_map=0` to reproduce the classical baseline (BASE configuration).

The full launch sequence for a complete experiment:

```bash
roslaunch turtlebot3_gazebo houseexpo.launch
roslaunch turtlebot3_slam turtlebot3_slam.launch
roslaunch turtlebot3_navigation move_base.launch
roslaunch explore_lite explore.launch
# For MC only:
roslaunch map_completion batch_map_completion_unet.launch
```

### Step 6: Analyse Exploration Logs

```bash
unzip exploration_logs_20260315.zip
```

Contains timestamped CSV logs for all 8 HouseExpo evaluation environments across BASE and MC configurations (3 runs each). Logs record elapsed time to reach coverage milestones: 70%, 75%, 80%, 85%, 90%, 95%, and maximum achieved coverage.

### Step 7: Compute Environment Metrics and Reproduce Figures

```bash
unzip env_classifier_and_figure_generator.zip
cd env_classifier_and_figure_generator
pip install matplotlib pillow numpy opencv-python-headless
```

Edit the `MAP_DIR` and `OUTPUT_DIR` variables at the top of
`generate_thesis_figures_final.py` to point to your local paths:

```python
MAP_DIR    = "/path/to/your/maps"   # directory containing .pgm and map_*.png files
OUTPUT_DIR = "/path/to/output"      # figures will be saved here
```

Then run:

```bash
python3 generate_thesis_figures_final.py
```

This script automatically calls `compute_env_metrics.py` to compute
structural properties from the ground-truth PGM files, assigns
difficulty tiers via a four-rule heuristic, and generates all 8
system-level evaluation figures (PDF and PNG).

To inspect metrics or generate the LaTeX table independently:

```bash
# All 8 environments:
python3 compute_env_metrics.py

# Single map:
python3 compute_env_metrics.py path/to/map.pgm
```

**Computed metrics per environment:**

| Metric | Description |
|--------|-------------|
| Free area (m²) | Navigable area = n\_free × (0.1 m)² |
| Narrow-corridor (%) | Free space lost after 5-pixel morphological erosion (radius = 5 px = 0.5 m) |
| Wall density (%) | Obstacles / (free + obstacles) |
| Unknown fraction (%) | Unknown pixels / total pixels |
| Openness (%) | 100 − narrow-corridor ratio |

**Difficulty tier assignment (four-rule heuristic):**

| Rule | Condition | Tier |
|------|-----------|------|
| 1 | `free_m2 < 600` AND `narrow < 35%` AND `wall < 8%` | Easy |
| 2 | `narrow ≥ 35%` OR `wall ≥ 8%` OR `openness ≤ 63%` | Hard |
| 3 | `free_m2 > 1500` AND `narrow > 25%` | Hard |
| 4 | `narrow < 20%` AND `wall < 5%` AND `openness > 80%` | Easy |
| default | — | Medium |

Rule 1 handles small environments where high narrow-corridor ratios reflect compact
doorway transitions rather than long passages (e.g. `46155736`, 406 m²).
Rule 3 handles large environments where moderate narrow corridors connect spatially
distant clusters, making navigation errors costly (e.g. `dd3b9ef6`, 1819 m²).
All 8 evaluation environments are correctly classified (verified against visual
floor plan inspection).

**Generated figures:**

| File | Content |
|------|---------|
| `fig1_heatmap_avantaj.pdf/png` | MC time advantage heatmap (environment × milestone) |
| `fig2_bar_milestone_sure.pdf/png` | Average exploration time by milestone |
| `fig3_lines_per_ortam_sure.pdf/png` | Per-environment time profiles |
| `fig4_lines_per_ortam_mesafe.pdf/png` | Per-environment distance profiles |
| `fig5_scatter_zorluk_avantaj.pdf/png` | Difficulty tier vs. MC advantage scatter |
| `fig6_pvalue_heatmap.pdf/png` | Statistical significance map (paired t-test) |
| `fig7_radar_ortam_profil.pdf/png` | Structural profile radar charts |
| `env_maps_8.pdf/png` | Ground-truth maps of all 8 environments |

The structural metrics in Table 2 and all system-level figures in Section VI
are generated by these scripts and are fully reproducible from the PGM map files.

---

## System Requirements

- ROS Noetic
- Gazebo 11
- Python 3.8+
- PyTorch (see `requirements` files inside each zip)
- TurtleBot3 ROS packages
- Docker (recommended for ROS environment isolation)

---

## Model Checkpoints and Metric Results

Pre-trained model checkpoints and evaluation metric results for all models are available via Google Drive:

- **Model Checkpoints and Metric Results**: [Google Drive](https://drive.google.com/drive/folders/1YEGeQQo2xxwP3Ltl-r_w7oUQwkD8cWol?usp=drive_link)

> **Note:** These links will be migrated to a permanent repository (Zenodo) upon publication.

---

## Dataset

The frontier-centered patch datasets are constructed from the [HouseExpo](https://github.com/TeaganLi/HouseExpo) indoor floor plan dataset:

> T. Li et al., "HouseExpo: A large-scale 2D indoor layout dataset for learning-based algorithms on mobile robots," in *Proc. IEEE/RSJ Int. Conf. Intell. Robot. Syst. (IROS)*, 2020, pp. 5839–5846.

| Dataset | Description | Link |
|---------|-------------|------|
| `dataset_patches` | Main frontier-centered 32×32 patch dataset (SDS/BDS) | [Google Drive](https://drive.google.com/file/d/1rV2ugK4UFYJP9R7uqv4WzI-5I_keTrD5/view?usp=drive_link) |
| `dataset_patches_64` | 64×64 patch dataset (ablation) | [Google Drive](https://drive.google.com/file/d/18io9o68RZAzHbRWMLi9sOX0H9SZZkPqB/view?usp=drive_link) |
| `dataset_patches_full` | Full patch pool including original 1,160-patch test set | [Google Drive](https://drive.google.com/file/d/1kuCWd1IrBpl13LxG6Ez3jB7UlpmzxQD3/view?usp=drive_link) |

> **Note:** These links will be migrated to a permanent repository (Zenodo) upon publication.

If any links are unavailable, please contact the corresponding author:
- selma.guzel@std.yildiz.edu.tr
- selmaguzelben@gmail.com

---

## Related Repositories

- [houseexpo_based_custom_dataset_with_gazebo](https://github.com/f0120050/houseexpo_based_custom_dataset_with_gazebo) — Gazebo environment generation from HouseExpo floor plans

---

## Citation

Citation will be added upon publication.

---

> **Note:** This repository is currently in its initial upload state and will be reorganized and documented in more detail in a future update.
