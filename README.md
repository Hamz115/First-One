# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GraspGen is a modular framework for diffusion-based 6-DOF robotic grasp generation that scales across diverse settings including different gripper types (industrial pinch gripper, suction), observability conditions (partial vs. complete 3D point clouds), and complexity (single-object vs. clutter grasping).

## Development Commands

### Installation

```bash
# Docker installation (recommended for training)
bash docker/build.sh

# Pip installation (for inference only)
pip install -e .
cd pointnet2_ops && pip install -e .
pip install pyrender && pip install PyOpenGL==3.1.5 transformers pyrender diffusers==0.11.1 timm huggingface-hub==0.25.2 scene-synthesizer[recommend]
```

### Docker Usage

```bash
# For inference with models
bash docker/run.sh <path_to_graspgen_code> --models <path_to_models_repo>

# For training with datasets
bash docker/run.sh <path_to_graspgen_code> --grasp_dataset <path_to_grasp_dataset> --object_dataset <path_to_object_dataset> --results <path_to_results>

# Start MeshCat server for visualization
meshcat-server  # or: bash docker/run_meshcat.sh
```

### Training

```bash
# Train generator model
cd /code && bash runs/train_graspgen_robotiq_2f_140_gen.sh

# Train discriminator model  
cd /code && bash runs/train_graspgen_robotiq_2f_140_dis.sh
```

### Running Tests

```bash
# Run all tests with pytest
cd /code && python3 -m pytest tests/

# Run specific test file
cd /code && python3 -m pytest tests/test_rotation_conversions.py
```

### Inference Demos

```bash
# Scene point cloud inference
cd /code && python scripts/demo_scene_pc.py --sample_data_dir /models/sample_data/real_scene_pc --gripper_config /models/checkpoints/graspgen_robotiq_2f_140.yml

# Object point cloud inference  
cd /code && python scripts/demo_object_pc.py --sample_data_dir /models/sample_data/real_object_pc --gripper_config /models/checkpoints/graspgen_robotiq_2f_140.yml

# Object mesh inference
cd /code && python scripts/demo_object_mesh.py --mesh_file /models/sample_data/meshes/box.obj --mesh_scale 1.0 --gripper_config /models/checkpoints/graspgen_robotiq_2f_140.yml
```

## Code Architecture

### Core Module Structure

- `grasp_gen/models/` - Neural network models including diffusion generator and discriminator
  - `generator.py` - Main diffusion-based grasp generation model
  - `discriminator.py` - Grasp scoring and ranking model
  - `grasp_gen.py` - Combined grasp generation pipeline
  - `ptv3/` - PointTransformer V3 backbone implementation

- `grasp_gen/dataset/` - Dataset loading and processing
  - Data generation for different gripper types (suction, antipodal)
  - Point cloud processing and augmentation
  - HDF5 caching for efficient training

- `grasp_gen/utils/` - Utility functions for rotation conversions, math operations, and visualization

- `scripts/` - Main training and inference scripts
  - `train_graspgen.py` - Unified training script for both generator and discriminator
  - `inference_graspgen.py` - Core inference pipeline
  - `demo_*.py` - Demo scripts for different input types

### Key Training Pipeline Details

1. **Dataset Caching**: Before training begins, the system builds an HDF5 cache of the dataset. This is handled automatically by `train_graspgen.py` - if cache doesn't exist, it builds it then continues with training.

2. **Multi-GPU Support**: Training scripts use PyTorch DDP for distributed training. Set `NGPU` environment variable in training scripts.

3. **Gripper Convention**: GraspGen uses a specific coordinate convention where:
   - Approach direction is positive Z-axis
   - Finger closing direction is along X-axis
   - Grippers are defined by YAML config, Python config, and URDF

### Important Configuration Files

- `runs/train_graspgen_*.sh` - Training scripts with hyperparameters for different grippers
- `config/` - Hydra configuration files for training and inference
- `assets/` - Gripper URDFs and configuration files

### Dataset Format

The project uses two main datasets:
1. **Grasp Dataset**: Pre-generated grasps for objects (57M+ grasps)
2. **Object Dataset**: 3D object meshes from Objaverse XL

Datasets follow specific formats documented in `docs/GRASP_DATASET_FORMAT.md` and `docs/GRIPPER_DESCRIPTION.md`.

## Notes

- CUDA 12.8 currently has issues with PTV3 backbone - use PointNet++ backbone instead
- On-generator training for discriminator provides best grasp scoring performance
- Training convergence: Generator ~3K epochs (40hrs on 8xA100), Discriminator ~3K epochs (90hrs on 8xA100)
- Monitor `reconstruction/error_trans_l2` for generator and validation AP score for discriminator
