# Multi-Object Grasp Generation Pipeline Documentation

## Overview
This document explains how to generate robotic grasps for multiple objects in a single scene using GraspGen.

## The Challenge
GraspGen is designed to process ONE object at a time. When you have a scene with multiple objects (like a basket with items inside), you need a strategy to:
1. Separate each object
2. Generate grasps for each individually
3. Combine results

## Input Requirements

### 1. RGB Image (RGB2.png)
- Standard color image from your camera
- Shows all objects in the scene

### 2. Depth Image (DEPTH.png)
- 16-bit depth map
- Each pixel value = distance from camera (usually in millimeters)
- Used to convert 2D pixels to 3D points

### 3. Segmentation Mask (final_filtered_segmentation_map.png)
- **CRITICAL**: Each object has a unique ID number
- Background = 0
- Object 1 = some number (e.g., 200)
- Object 2 = different number (e.g., 85)
- etc.

## The Pipeline

### Stage 1: 2D to 3D Conversion

```
RGB Image + Depth Image → 3D Point Cloud
```

The math (simplified):
```python
# For each pixel:
3D_point.z = depth_value / 1000  # Convert to meters
3D_point.x = (pixel_x - camera_center_x) * z / focal_length
3D_point.y = (pixel_y - camera_center_y) * z / focal_length
3D_point.color = RGB_image[pixel_x, pixel_y]
```

### Stage 2: Object Extraction

```
Full Point Cloud + Segmentation Mask → Individual Object Point Clouds
```

For each unique ID in the mask:
1. Find all pixels with that ID
2. Extract corresponding 3D points
3. Save as separate point cloud

Example from your scene:
- Object ID 200 → 2,468 pixels → 1,680 3D points (basket)
- Object ID 85 → 1,650 pixels → ~1,200 3D points (yellow object)

### Stage 3: GraspGen Processing Loop

```
FOR each object:
    object_points → GraspGen → grasps + scores
```

The loop:
```python
for object_id in [200, 85, 212, 134, 72, 38]:
    # 1. Load object point cloud
    points = load_object(object_id)
    
    # 2. Run GraspGen
    grasps, scores = GraspGen.generate(points, num_grasps=100)
    
    # 3. Filter by quality
    good_grasps = grasps[scores > 0.7]
    
    # 4. Take top 20
    top_grasps = good_grasps[:20]
    
    # 5. Save results
    save_grasps(object_id, top_grasps)
```

### Stage 4: Results Combination

```
Individual Grasps → Combined Grasp Set
```

All grasps are combined with tracking:
```json
{
  "object_id": 200,
  "grasp": [4x4 transformation matrix],
  "score": 0.95,
  "grasp_idx": 0
}
```

## File Structure

### Input Files
```
test_scene/
├── RGB2.png                    # Color image
├── DEPTH.png                   # Depth map
└── final_filtered_segmentation_map.png  # Object masks
```

### Generated Files
```
custom_scenes_multi/
├── object_200.json             # Basket point cloud data
├── object_200_grasps.json      # Basket grasp poses
├── object_200_object.ply       # 3D visualization file
├── object_200_scene.ply        # Full scene for context
├── object_85.json              # Yellow object 1 data
├── object_85_grasps.json       # Yellow object 1 grasps
... (same pattern for each object)
└── all_grasps_combined.json    # All 216 grasps together
```

## Scripts Explained

### 1. process_custom_scene.py
- **Purpose**: Convert RGB-D to point cloud for ONE object
- **Input**: RGB, Depth, Mask, Object ID
- **Output**: JSON file in GraspGen format

### 2. process_all_objects.py
- **Purpose**: Automate processing for ALL objects
- **Process**: 
  1. Find all object IDs in mask
  2. Call process_custom_scene.py for each
  3. Run GraspGen on each
  4. Combine results

### 3. visualize_all_grasps.py
- **Purpose**: Show all objects and grasps together
- **Features**:
  - Color-codes each object
  - Shows top grasps per object
  - Quality-based coloring

## Results from Your Scene

| Object ID | Description | Pixels | 3D Points | Grasps Generated | Best Score |
|-----------|------------|--------|-----------|------------------|------------|
| 200 | Basket | 2,468 | 1,680 | 40 | 0.949 |
| 85 | Yellow Object 1 | 1,650 | ~1,200 | 40 | 0.942 |
| 212 | Object 2 | 1,079 | ~800 | 40 | 0.949 |
| 134 | Small Object | 437 | ~300 | 6 | 0.758 |
| 72 | Small Object | 346 | ~250 | 50 | 0.805 |
| 38 | Tiny Object | 177 | ~120 | 40 | 0.879 |

**Total: 216 grasps across 6 objects**

## Key Concepts

### Why Not Process Everything at Once?
GraspGen's neural network is trained on single objects. It:
- Expects one coherent object point cloud
- Generates grasps for that specific geometry
- Cannot distinguish between multiple objects

### Why Use Segmentation?
Without segmentation, you'd only get grasps for the dominant object (usually the largest). Segmentation lets you:
- Process each object optimally
- Know which grasp belongs to which object
- Handle cluttered scenes

### Grasp Quality Scores
- **0.9-1.0**: Excellent grasp (green in visualization)
- **0.8-0.9**: Good grasp (yellow)
- **0.7-0.8**: Acceptable (orange)
- **<0.7**: Rejected (not shown)

## Running the Complete Pipeline

### Quick Start (Your Test Scene)
```bash
# Process all objects and generate grasps
docker exec graspgen-main /bin/bash -c "cd /code && python process_all_objects.py"

# Visualize results
docker exec graspgen-main /bin/bash -c "cd /code && python visualize_all_grasps.py"

# View at: http://localhost:7000/static/
```

### For New Scenes
1. Capture RGB, Depth, and Segmentation mask
2. Place in a folder
3. Modify paths in process_all_objects.py
4. Run the pipeline

## Troubleshooting

### "No points found for object ID X"
- Check mask values match what you specify
- Ensure object is visible in depth image
- Verify depth values are valid (not 0)

### Too few grasps generated
- Object might be too small (<100 points)
- Lower grasp_threshold (try 0.6)
- Increase num_grasps parameter

### Grasps look wrong
- Check camera intrinsics are correct
- Verify depth_scale (1000 for mm, 1 for m)
- Ensure point cloud looks correct in PLY files

## Advanced Options

### Process Specific Objects Only
```python
# In process_all_objects.py, modify:
object_ids = [200, 85]  # Only process basket and first object
```

### Change Gripper Type
```python
gripper_config = "/models/checkpoints/graspgen_franka_panda.yml"  # or suction
```

### Adjust Grasp Generation
```python
num_grasps=200  # Generate more candidates
top_k=30       # Keep more grasps per object
grasp_threshold=0.6  # Lower quality threshold
```

## Summary
This pipeline solves the multi-object grasping problem by:
1. Using segmentation to identify individual objects
2. Processing each object separately through GraspGen
3. Combining results for comprehensive scene grasping
4. Maintaining object-grasp associations for execution

The result: High-quality, object-specific grasps for complex scenes with multiple graspable items.
