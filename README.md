# Automated Pores Generation

## 📖 Introduction
This project provides a fully automated method to generate 3D porous structures (with spherical or elliptical pores) inside a cubic volume using FreeCAD.
It uses Gaussian distribution to randomly distribute pores and avoids overlaps between them.

Outputs are STL models and cross-sectional PNG images for visualization and further simulations.

## ⚙️ Software Requirements
| Tool | Version | Purpose |
|------|---------|---------|
| FreeCAD | 0.19 or newer | CAD modeling, geometry creation |
| Python | 3.8 or 3.9 recommended | Running the notebook |
| Jupyter Notebook | Latest | Executing the notebook steps |

**Important**: FreeCAD must be properly installed and its Python modules accessible.

## 🛠️ Full Installation Guide

### Step 1: Install FreeCAD
- Download from [FreeCAD Official Site](https://www.freecadweb.org/downloads.php).
- Install FreeCAD.
- (Optional) Add the FreeCAD/bin directory to your system PATH.

### Step 2: Set up Python Environment
**Option A**: Use FreeCAD's internal Python console.

**Option B (Recommended)**: Create a Conda environment:
```bash
conda create -n freecad-env python=3.8
conda activate freecad-env
conda install notebook
```

Link FreeCAD's libraries inside your script:
```python
import sys
sys.path.append('C:/Program Files/FreeCAD 0.20/bin')  # Example path
```

### Step 3: Install Jupyter Extensions (optional)
```bash
conda install -c conda-forge jupyter_contrib_nbextensions
```

## 🚀 How to Run the Notebook
1. Launch Jupyter:
```bash
jupyter notebook
```

2. Open `Automated_pores_generation.ipynb`.

3. Run each cell step-by-step:
   - Import libraries
   - Define generation functions
   - Set parameters
   - Call generate_models(...)

4. Outputs (.stl + cross-section .png) will be saved automatically.

## 🔥 Detailed Algorithm Explanation

### 1. Spherical Pores Generation
| Step | Description |
|------|-------------|
| 1. | Set the number of pores `num_pores`, mean and standard deviation for Gaussian distribution. |
| 2. | Randomly generate (x, y, z) positions inside the cube based on the Gaussian parameters. |
| 3. | Ensure that each new pore center is at least 2 × radius + gap away from all previous pores to avoid overlaps. |
| 4. | If placement fails after 100 attempts, accept the partial configuration. |
| 5. | Create a spherical pore at each valid (x, y, z) using Part.makeSphere(radius). |
| 6. | Export the final 3D model as .stl and a cross-section .png. |

**Summary:**
➔ Random Gaussian placement + Overlap avoidance + Sphere creation + Export

### 2. Elliptical Pores Generation
(Currently partially implemented — ready for extension.)

| Step | Description |
|------|-------------|
| 1. | Similar random (x, y, z) placement using Gaussian distribution. |
| 2. | Instead of simple spheres, ellipsoids (elongated spheres) are generated. |
| 3. | Use Part.makeEllipsoid(rx, ry, rz) or scale spheres along different axes to simulate elliptical pores. |
| 4. | Apply random rotation to ellipsoids for a more realistic distribution (rotation matrices or Euler angles). |
| 5. | Ensure non-overlapping based on a bounding box approximation (more complex than spheres). |
| 6. | Save final model as .stl and cross-sectional image .png. |

**Note:**
➔ Elliptical pore generation code is partly present in the notebook but needs minor additions to fully randomize orientations.

## 🏗️ Output Structure
Outputs are automatically saved into your selected folder.

Example:
```
output_folder/
├── model_1.stl
├── model_1_cross_section.png
├── model_2.stl
├── model_2_cross_section.png
└── ...
```

## ✍️ Key Functions Explained
`generate_gaussian_configuration(...)`
- Random 3D position generation with overlap checks.

`generate_models(...)`
- Full pipeline for creating 3D models from configuration to export.

## 🎯 Quick Example Usage
```python
generate_models(
    num_pores=50,
    model_count=10,
    base_dir="output_folder",
    mean=0.5,
    std_dev=0.15,
    min_distance=0.02,
    cube_size=10,
    radius=0.5,
    gap=0.1
)
```

## ⚡ Important Tips
- Smaller gaps or bigger radii increase placement difficulty — adjust accordingly.
- For elliptical pores, extra collision checking might be needed if rotation is introduced.
- Gaussian parameters control how densely pores are centered in the cube.

## 📢 Final Note
This notebook provides a flexible, automated workflow for creating porous structures inside a 3D cube — useful for material science, AI simulations, or mechanical modeling.

With minor extensions, elliptical pore randomization can be fully enabled too! 🚀
