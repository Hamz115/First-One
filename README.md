# GraspGen Complete Setup and Understanding Guide

## What is GraspGen?

GraspGen is a modular framework for **diffusion-based 6-DOF robotic grasp generation**. It uses AI to predict where and how a robot should grasp objects in 3D space.

### Key Capabilities:
- **Multiple Gripper Types**: Works with industrial pinch grippers and suction grippers
- **Flexible Input**: Handles both partial and complete 3D point clouds
- **Complex Scenarios**: Can grasp single objects or objects in cluttered scenes
- **6-DOF Grasps**: Generates full 6 degrees-of-freedom grasps (3D position + 3D orientation)

## System Requirements

- **GPU**: NVIDIA GPU with CUDA support (8GB+ VRAM recommended)
- **CUDA**: Version 11.x or 12.x (Note: CUDA 12.8 has issues with PTV3 backbone)
- **Python**: 3.8 or higher
- **OS**: Linux (Ubuntu 20.04/22.04 recommended)
- **RAM**: 16GB+ recommended
- **Storage**: ~50GB for models and datasets

## Installation Options

You have two main installation paths:

### Option 1: Docker Installation (Recommended for Training)
Best if you want to train models or have a clean, isolated environment.

### Option 2: Pip Installation (For Inference Only)
Best if you just want to run pre-trained models for grasp generation.

---

## Step-by-Step Setup Guide

### Step 1: Clone the Repository

```bash
# If you haven't already cloned it
git clone https://github.com/NVlabs/GraspGen.git
cd GraspGen
```

### Step 2: Download Pre-trained Models

```bash
# Download the pre-trained models (required for inference)
# These include checkpoints for different gripper types
wget https://huggingface.co/datasets/changhaonan/GraspGen/resolve/main/models.tar.gz
tar -xzf models.tar.gz
```

### Step 3: Choose Your Installation Method

## Option A: Docker Installation (Recommended)

### Step 3A.1: Install Docker and NVIDIA Container Toolkit

```bash
# Install Docker
sudo apt-get update
sudo apt-get install docker.io
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to docker group
sudo usermod -aG docker $USER
# Log out and back in for this to take effect

# Install NVIDIA Container Toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

### Step 3A.2: Build Docker Image

```bash
# From the GraspGen directory
bash docker/build.sh
```

### Step 3A.3: Run Docker Container

For inference only:
```bash
bash docker/run.sh $(pwd) --models $(pwd)/models
```

For training (if you have datasets):
```bash
bash docker/run.sh $(pwd) \
    --models $(pwd)/models \
    --grasp_dataset /path/to/grasp_dataset \
    --object_dataset /path/to/object_dataset \
    --results $(pwd)/results
```

## Option B: Pip Installation (Inference Only)

### Step 3B.1: Create Python Environment

```bash
# Create conda environment (recommended)
conda create -n graspgen python=3.9
conda activate graspgen

# Or use venv
python3 -m venv graspgen_env
source graspgen_env/bin/activate
```

### Step 3B.2: Install PyTorch

```bash
# Install PyTorch with CUDA support (adjust CUDA version as needed)
# For CUDA 11.8
pip install torch==2.0.1 torchvision==0.15.2 torchaudio==2.0.2 --index-url https://download.pytorch.org/whl/cu118

# For CUDA 12.1
pip install torch==2.1.0 torchvision==0.16.0 torchaudio==2.1.0 --index-url https://download.pytorch.org/whl/cu121
```

### Step 3B.3: Install GraspGen

```bash
# Install main package
pip install -e .

# Install PointNet++ operations
cd pointnet2_ops
pip install -e .
cd ..

# Install additional dependencies
pip install pyrender PyOpenGL==3.1.5 transformers diffusers==0.11.1 timm huggingface-hub==0.25.2 scene-synthesizer[recommend]

# Install visualization tools
pip install meshcat trimesh
```

### Step 4: Verify Installation

```bash
# Test import
python -c "import grasp_gen; print('GraspGen imported successfully')"

# Run tests
python -m pytest tests/test_rotation_conversions.py -v
```

### Step 5: Start Visualization Server (Optional)

For 3D visualization of grasps:
```bash
# In a separate terminal
meshcat-server

# Or use Docker
bash docker/run_meshcat.sh
```

Open browser at `http://localhost:7000/static/`

---

## Running Your First Demo

### Demo 1: Object Point Cloud Grasp Generation

```bash
# Navigate to GraspGen directory
cd /path/to/GraspGen

# Run demo with sample data
python scripts/demo_object_pc.py \
    --sample_data_dir models/sample_data/real_object_pc \
    --gripper_config models/checkpoints/graspgen_robotiq_2f_140.yml
```

### Demo 2: Scene Point Cloud Grasp Generation

```bash
python scripts/demo_scene_pc.py \
    --sample_data_dir models/sample_data/real_scene_pc \
    --gripper_config models/checkpoints/graspgen_robotiq_2f_140.yml
```

### Demo 3: Object Mesh Grasp Generation

```bash
python scripts/demo_object_mesh.py \
    --mesh_file models/sample_data/meshes/box.obj \
    --mesh_scale 1.0 \
    --gripper_config models/checkpoints/graspgen_robotiq_2f_140.yml
```

---

## Understanding the Output

When you run a demo, GraspGen will:
1. Load the input (point cloud or mesh)
2. Generate multiple grasp candidates using diffusion
3. Score and rank grasps using the discriminator
4. Visualize the top grasps (if MeshCat is running)
5. Save results to `results/` directory

### Output Files:
- `grasps.npz`: Generated grasp poses (4x4 transformation matrices)
- `scores.txt`: Grasp quality scores
- `visualization.html`: 3D visualization (if enabled)

---

## Troubleshooting

### Common Issues:

1. **CUDA Out of Memory**
   - Reduce batch size in inference scripts
   - Use smaller point cloud samples

2. **ImportError for grasp_gen**
   - Make sure you're in the correct environment
   - Verify installation with `pip list | grep grasp`

3. **MeshCat not showing visualization**
   - Ensure meshcat-server is running
   - Check firewall settings for port 7000

4. **Docker permission denied**
   - Run `sudo usermod -aG docker $USER` and re-login

5. **CUDA version mismatch**
   - Check CUDA version: `nvidia-smi`
   - Install matching PyTorch version

---

## Next Steps

1. **Try Different Grippers**: Change `--gripper_config` to test other gripper types
2. **Use Your Own Data**: Prepare point clouds or meshes of your objects
3. **Train Custom Models**: Follow training guides in `docs/` directory
4. **Integrate with Robot**: Use generated grasps with your robot control system

---

## Quick Reference Commands

```bash
# Check GPU
nvidia-smi

# Check CUDA version
nvcc --version

# Monitor GPU usage during inference
watch -n 1 nvidia-smi

# List available gripper configs
ls models/checkpoints/*.yml

# View sample data
ls models/sample_data/
```

## Getting Help

- Check `docs/` directory for detailed documentation
- Review test files in `tests/` for usage examples
- Look at demo scripts in `scripts/` for implementation patterns
