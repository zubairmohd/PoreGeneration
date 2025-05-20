# Automated Pores Generation

## 📖 Introduction
This project provides a fully automated method to generate 3D porous structures (with spherical or elliptical pores) inside a cubic volume using FreeCAD.
It uses Gaussian distribution to randomly distribute pores and avoids overlaps between them.

Outputs are STL and STEP models and cross-sectional PNG images for visualization and further simulations.
### Key Features
- **Fully Automated Generation**: Generate multiple porous models with a single command
- **Customizable Porosity**: Control pore size, distribution, and density
- **Multiple Pore Types**: Support for both spherical and elliptical pores
- **Cross-Sectional Analysis**: Generate detailed 2D slices to visualize internal structure
- **FreeCAD Integration**: Leverages powerful CAD capabilities with Python scripting
- **STL Export**: Ready for 3D printing or further simulation

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

## 🔥 Detailed Algorithm Explanation and Code

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

#### Spherical Pores Implementation

```python
import FreeCAD as App
import FreeCADGui as Gui
import Part
import random
import os
import math

def generate_gaussian_configuration(num_pores, mean, std_dev, cube_size, radius, gap):
    configuration = []
    d_min = 2 * radius + gap  # Minimum distance between pore centers

    for _ in range(num_pores):
        attempts = 0
        while attempts < 100:
            x = random.gauss(mean, std_dev) * cube_size
            y = random.gauss(mean, std_dev) * cube_size
            z = random.gauss(mean, std_dev) * cube_size

            # Check boundaries
            if not (radius <= x <= cube_size - radius and
                    radius <= y <= cube_size - radius and
                    radius <= z <= cube_size - radius):
                attempts += 1
                continue

            new_center = App.Vector(x, y, z)
            if all(new_center.distanceToPoint(App.Vector(*c)) >= d_min for c in configuration):
                configuration.append((x, y, z))
                break
            attempts += 1

        if attempts >= 100:
            print("Failed to place a pore after 100 attempts.")
            return None

    return configuration

def calculate_pore_volume(radius, num_pores):
    return num_pores * (4 / 3) * math.pi * radius**3

def save_views(view, output_dir, base_name):
    orientations = {
        "top": view.viewTop,
        "front": view.viewFront,
        "right": view.viewRight,
        "isometric": view.viewIsometric
    }
    for name, func in orientations.items():
        func()
        view.fitAll()
        path = os.path.join(output_dir, f"{base_name}_{name}.png")
        view.saveImage(path, 1024, 1024, 'Transparent')

def generate_cross_sections(obj, plane, cube_size, output_dir, model_num, step_size=0.1):
    offset = 0
    while offset <= cube_size:
        if plane == "XY":
            section_plane = Part.makePlane(cube_size, cube_size, App.Vector(0, 0, offset), App.Vector(0, 0, 1))
            view_dir = App.Vector(0, 0, 1)
        elif plane == "XZ":
            section_plane = Part.makePlane(cube_size, cube_size, App.Vector(0, offset, 0), App.Vector(0, 1, 0))
            view_dir = App.Vector(0, 1, 0)
        else:
            raise ValueError("Unsupported plane")

        section_shape = obj.Shape.common(section_plane)
        if not section_shape.isNull():
            section_obj = App.ActiveDocument.addObject("Part::Feature", f"Section_{plane}_{offset:.1f}")
            section_obj.Shape = section_shape

            # Hide all other objects to avoid shadows
            for o in App.ActiveDocument.Objects:
                o.ViewObject.Visibility = False
            section_obj.ViewObject.Visibility = True

            App.ActiveDocument.recompute()

            # Set orthographic view directly along slicing direction
            view = Gui.ActiveDocument.ActiveView
            view.setCameraType("Orthographic")
            view.setViewDirection(view_dir)
            view.fitAll()

            # Save clean image
            screenshot_path = os.path.join(output_dir, f"cross_section_{plane}_{offset:.1f}_model_{model_num}.png")
            view.saveImage(screenshot_path, 1024, 1024, 'White')  # White background avoids transparency artifacts

            # Cleanup
            App.ActiveDocument.removeObject(section_obj.Name)

        offset += step_size


def generate_models(
    num_pores=30,
    model_count=2,
    base_dir=r"D:\PHD\Pores\new",
    mean=0.5,
    std_dev=0.15,
    cube_size=1.0,  # mm
    radius=0.04,    # mm
    gap=0.01,
    solid_density=8.96  # g/cm³ for copper
):
    valid_models = 0

    while valid_models < model_count:
        print(f"\nGenerating model {valid_models + 1}")
        config = generate_gaussian_configuration(num_pores, mean, std_dev, cube_size, radius, gap)

        if config is None:
            print("Skipping due to failed configuration")
            continue

        folder = f"Model_{valid_models + 1}_R{radius}_P{num_pores}"
        model_dir = os.path.join(base_dir, folder)
        os.makedirs(model_dir, exist_ok=True)

        doc = App.newDocument(f"Model_{valid_models + 1}")
        cube = Part.makeBox(cube_size, cube_size, cube_size)

        for x, y, z in config:
            sphere = Part.makeSphere(radius, App.Vector(x, y, z))
            cube = cube.cut(sphere)

        obj = doc.addObject("Part::Feature", "CubeWithPores")
        obj.Shape = cube
        obj.ViewObject.Transparency = 70
        App.ActiveDocument.recompute()

        # Export STEP file
        step_path = os.path.join(model_dir, f"model_{valid_models + 1}.step")
        Part.export([obj], step_path)
        print(f"Saved STEP file: {step_path}")

        # Porosity calculation
        V_cube = cube_size ** 3  # mm³
        V_pores = calculate_pore_volume(radius, num_pores)
        porosity = V_pores / V_cube
        solid_density_mm3 = solid_density / 1000  # g/mm³
        effective_density = solid_density_mm3 * (1 - porosity)
        mass = effective_density * V_cube

        with open(os.path.join(model_dir, "report.txt"), "w") as f:
            f.write(f"Porosity: {porosity*100:.2f}%\n")
            f.write(f"Effective Density: {effective_density:.6f} g/mm³\n")
            f.write(f"Mass: {mass:.6f} g\n")

        # Views and cross-sections
        if Gui.ActiveDocument:
            view = Gui.ActiveDocument.ActiveView
            save_views(view, model_dir, f"model_{valid_models + 1}")
            generate_cross_sections(obj, "XY", cube_size, model_dir, valid_models + 1)
            generate_cross_sections(obj, "XZ", cube_size, model_dir, valid_models + 1)

        App.closeDocument(doc.Name)
        valid_models += 1

    print("✅ All models generated.")

# Run the function
generate_models()

```

### 2. Elliptical Pores Generation

| Step | Description |
|------|-------------|
| 1. | Similar random (x, y, z) placement using Gaussian distribution. |
| 2. | Instead of simple spheres, ellipsoids (elongated spheres) are generated. |
| 3. | Use Part.makeEllipsoid(rx, ry, rz) or scale spheres along different axes to simulate elliptical pores. |
| 4. | Apply random rotation to ellipsoids for a more realistic distribution (rotation matrices or Euler angles). |
| 5. | Ensure non-overlapping based on a bounding box approximation (more complex than spheres). |
| 6. | Save final model as .stl and cross-sectional image .png. |

#### Elliptical Pores Implementation

```python
import FreeCAD as App
import Part
import random
import os
import FreeCADGui as Gui
import Mesh

def generate_gaussian_configuration(num_pores, mean, std_dev, min_distance, cube_size, major_axis, minor_axis, gap):
    """
    Generate a Gaussian configuration for elliptical pore centers without overlap.
    """
    configuration = []
    for _ in range(num_pores):
        attempts = 0
        while True:
            x = round(random.gauss(mean, std_dev), 4) * cube_size
            y = round(random.gauss(mean, std_dev), 4) * cube_size
            z = round(random.gauss(mean, std_dev), 4) * cube_size

            # Ensure the ellipsoid fits within the cube boundaries with the required gap
            if (major_axis / 2 + gap <= x <= cube_size - major_axis / 2 - gap) and \
               (minor_axis / 2 + gap <= y <= cube_size - minor_axis / 2 - gap) and \
               (minor_axis / 2 + gap <= z <= cube_size - minor_axis / 2 - gap):
                new_center = (x, y, z)

                # Ensure no intersection and apply a minimum distance between pores
                if all(App.Vector(*new_center).distanceToPoint(App.Vector(*center)) >= min_distance for center in configuration):
                    configuration.append(new_center)
                    break

            attempts += 1
            if attempts > 100:  # To avoid infinite loops, limit the attempts
                return None  # Return None if a valid configuration can't be found

    return configuration

def capture_images(view, folder_path, base_name):
    """
    Save isometric, front, top, and right views of the model.
    """
    views = [
        ("isometric", "iso"),
        ("front", "front"),
        ("top", "top"),
        ("right", "right"),
    ]

    for view_name, suffix in views:
        if view_name == "isometric":
            view.viewIsometric()
        elif view_name == "front":
            view.viewFront()
        elif view_name == "top":
            view.viewTop()
        elif view_name == "right":
            view.viewRight()

        view.fitAll()
        image_path = os.path.join(folder_path, f"{base_name}_{suffix}.png")
        view.saveImage(image_path, 1024, 1024, 'Transparent')

def generate_cross_section_images(obj, plane, cube_size, folder_path, model_name, step_size=0.1):
    """
    Generate and save cross-sectional images for the given model along the specified plane.
    """
    offset = 0
    while offset <= cube_size:
        if plane == "XY":
            section_plane = Part.makePlane(cube_size, cube_size, App.Vector(0, 0, offset), App.Vector(0, 0, 1))
        elif plane == "XZ":
            section_plane = Part.makePlane(cube_size, cube_size, App.Vector(0, offset, 0), App.Vector(0, 1, 0))
        else:
            raise ValueError("Invalid plane. Choose 'XY' or 'XZ'.")

        section_shape = obj.Shape.common(section_plane)
        if section_shape.isNull():
            offset += step_size
            continue

        section_obj = App.ActiveDocument.addObject("Part::Feature", f"{plane}_CrossSection_{offset:.1f}")
        section_obj.Shape = section_shape

        App.ActiveDocument.recompute()

        view = Gui.ActiveDocument.ActiveView
        view.viewIsometric()
        view.fitAll()
        if plane == "XY":
            view.viewTop()
        elif plane == "XZ":
            view.viewFront()

        image_path = os.path.join(folder_path, f"{model_name}_cross_section_{plane}_{offset:.1f}.png")
        view.saveImage(image_path, 1024, 1024, 'Transparent')

        App.ActiveDocument.removeObject(section_obj.Name)
        offset += step_size

def generate_models(num_pores, model_count, output_dir, mean, std_dev, min_distance, cube_size, major_axis, minor_axis, gap):
    """
    Generate models with elliptical pores, save STL files, and save cross-section images.
    """
    valid_models = 0
    while valid_models < model_count:
        config = generate_gaussian_configuration(num_pores, mean, std_dev, min_distance, cube_size, major_axis, minor_axis, gap)

        if config is None:
            continue

        doc = App.newDocument(f"Model_{valid_models + 1}")
        cube = Part.makeBox(cube_size, cube_size, cube_size)

        for x, y, z in config:
            ellipsoid = doc.addObject("Part::Ellipsoid", "Ellipsoid")
            ellipsoid.Radius1 = minor_axis / 2
            ellipsoid.Radius2 = minor_axis / 2
            ellipsoid.Radius3 = major_axis / 2
            ellipsoid.Placement = App.Placement(App.Vector(x, y, z), App.Rotation(0, 0, 0))
            doc.recompute()
            cube = cube.cut(ellipsoid.Shape)

        final_obj = doc.addObject("Part::Feature", "CubeWithEllipticalPores")
        final_obj.Shape = cube
        final_obj.ViewObject.Transparency = 70

        model_folder = os.path.join(output_dir, f"Model_Major_{major_axis}_Minor_{minor_axis}_StdDev_{std_dev}_Pores_{num_pores}_Model_{valid_models + 1}")
        os.makedirs(model_folder, exist_ok=True)
        base_name = f"model_{valid_models + 1}"

        # Save STL file
        export_path_stl = os.path.join(model_folder, f"{base_name}.stl")
        Mesh.export([final_obj], export_path_stl)

        # Save isometric, front, top, and right views
        view = Gui.ActiveDocument.ActiveView
        capture_images(view, model_folder, base_name)

        # Save cross-sectional images for XY and XZ planes
        generate_cross_section_images(final_obj, "XY", cube_size, model_folder, base_name, step_size=0.1)
        generate_cross_section_images(final_obj, "XZ", cube_size, model_folder, base_name, step_size=0.1)

        App.closeDocument(doc.Name)
        valid_models += 1
```

## 🔪 Cross-Section Generation Explained

The cross-section generation is a key feature of this tool, providing visualizations of the internal porous structure at different planes.

### What is a Cross-Section?
A cross-section is a 2D slice through the 3D model, allowing you to see the internal structure at a specific position. This is similar to cutting through an object with a knife and viewing the cut surface.

### Step Size Parameter
The `step_size` parameter (default: 0.1) controls:
- The distance between consecutive cross-section planes
- Smaller step size = more cross-sections = higher resolution view of internal structure
- Larger step size = fewer cross-sections = faster processing but less detailed view

For example, with a cube of size 1.0 and step_size of 0.1:
- The algorithm will create cross-sections at positions 0.0, 0.1, 0.2, ..., 0.9, 1.0
- This results in 11 cross-section images per plane

### Cross-Section Planes
The tool supports multiple cross-section planes:
- **XY plane**: Horizontal slices (top view)
- **XZ plane**: Front-to-back slices (front view)
- **YZ plane**: Side-to-side slices (side view)

### Cross-Section Processing
1. A plane is created at the specified position
2. The intersection between this plane and the 3D model is calculated
3. The result is rendered and saved as a PNG image
4. This process repeats for each position along the chosen axis

### Output Naming Convention
Cross-section images follow this naming pattern:
```
model_name_cross_section_[PLANE]_[POSITION].png
```
Example: `model_1_cross_section_XY_0.5.png` is a horizontal (XY) cross-section at the middle (0.5) of model 1.

## 🏗️ Output Structure
Outputs are automatically saved into your selected folder with organized directories for each model.

Example:
```
output_folder/
├── Model_Radius_0.50_StdDev_0.15_Pores_50_1/
│   ├── model_1.stl                             # 3D model file
│   ├── model_1_iso.png                         # Isometric view
│   ├── model_1_front.png                       # Front view
│   ├── model_1_top.png                         # Top view  
│   ├── model_1_right.png                       # Right view
│   ├── model_1_cross_section_XY_0.1.png        # XY cross-section at position 0.1
│   ├── model_1_cross_section_XY_0.2.png        # XY cross-section at position 0.2
│   ├── ...
│   ├── model_1_cross_section_XZ_0.1.png        # XZ cross-section at position 0.1
│   └── ...
├── Model_Radius_0.50_StdDev_0.15_Pores_50_2/
│   └── ...
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

## ⚡Further Steps 
- Once you have generated the CAD data with pores ----- >  next step is to use Ansys Script to Run the simulations
