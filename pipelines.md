# MRI-to-Mesh Pipelines for FEniCS

This document outlines the workflows used at Simula for converting raw MRI data into finite element meshes suitable for FEniCS/FEniCSx simulations. It covers current software choices, specific user pipelines, dataset locations, and future development needs.

## 1. Quick Reference: Current Workflows

A summary of who is using what, serving as a point of contact for specific variations of the pipeline.

| Contact | Meshing Engine | Segmentation Source | Dataset / Repo | Key Notes |
| :--- | :--- | :--- | :--- | :--- |
| **Ingvild** | Ftetwild | FreeSurfer / SynthSeg | [OpenNeuro ds004478](https://openneuro.org/datasets/ds004478) | Uses `mri2mesh`. Feature request: Node index tracking. |
| **Andreas** | SVMTK | FreeSurfer | [SVMTK Repo](https://github.com/SVMTK/SVMTK) | Standard SVMTK pipeline. |
| **Henrik** | Ftetwild | NeuroQuant | [Biophysics Repo](https://github.com/scientificcomputing/biophysics-anatomical-changes-mci/) | Uses a config-file driven approach. Handles NeuroQuant labels (different from FS). |
| **Cecile** | Ftetwild | FreeSurfer / SynthSeg | [CerebroSpinalFlow](https://github.com/cdaversin/CerebroSpinalFlow) | Based on [Marius Causemann's transport repo](https://github.com/MariusCausemann/brain-PVS-SAS-transport). |
| **Timo / Jørgen**| SVMTK / PyVista | FreeSurfer | [Gonzo](https://github.com/jorgenriseth/gonzo) | SVMTK preferred for surface preprocessing (keeps sulci better than PyVista). |

---

## 2. Pipeline Stages

### Stage 1: Raw Data to Images
Conversion of scanner output (typically DICOM) to research-standard formats (NIfTI).

* **Tools:** `dcm2niix` (Standard), proprietary scanner tools.
* **Target Modalities:**
    * **T1/T2:** Structural anatomy (geometry).
    * **Look-Locker:** T1 mapping.
    * **DTI:** Diffusion Tensor Imaging (anisotropy tensors).
    * **fMRI/BOLD:** Functional activation maps.
    * **FLAIR:** Fluid-attenuated inversion recovery.

### Stage 2: Registration
Aligning different image modalities (e.g., registering DTI to the T1 structural scan) or registering subjects to a common atlas (MNI).

* **[Greedy]([https://sites.google.com/view/greedy-reg/](https://www.itksnap.org/pmwiki/pmwiki.php?n=SourceCode.SourceCode)):** Fast, effective for standard rigid/affine registration.
* **[ANTs](http://stnava.github.io/ANTs/):** Gold standard for non-linear warping and complex registration.
* **[FSL FLIRT](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FLIRT):** Robust linear registration (often used in `mri2mesh` pipelines).

### Stage 3: Segmentation
Classifying voxels into tissue types (White Matter, Gray Matter, CSF).

#### A. FreeSurfer (Standard)
* **Command:** `recon-all`
* **Pros:** Industry standard, highly detailed cortical surfaces, separates hemispheres.
* **Cons:** Slow (hours), requires high-quality T1.
* **Output:** `aseg.mgz`, `lh.pial`, `rh.white`.

#### B. Deep Learning (SynthSeg)
* **Command:** `mri_synthseg`
* **Pros:** Fast (minutes), robust to domain shifts and lower resolution scans.
* **Cons:** Surfaces may lack the topological guarantees of FreeSurfer.


#### Other alternatives we migth want to consider
* [FastSurfer](https://github.com/Deep-MI/FastSurfer)
* [NextBrain](https://github-pages.ucl.ac.uk/NextBrain/#/home)
* [GOUHFI](https://arxiv.org/abs/2505.11445)
* [SAM3](https://ai.meta.com/sam3/)
* NeuroQuant - This is a commercial tool used by Anders Dale

### Stage 4: Surface Extraction
Converting voxel labels into triangular surface meshes (STL/PLY).

* **SVMTK (Recommended for complex geometry):**
    * Better at preserving deep sulci (valleys) which resolution-limited segmentations might smooth out.
    * Superior for boolean operations (e.g., patching holes in ventricles) compared to PyVista.
* **FreeSurfer:**
    * Native extraction via `mri_tessellate` or `mri_mc`.
* **PyVista:**
    * Uses VTK marching cubes. Good for visualization, but requires careful smoothing to avoid non-manifold artifacts in complex brain geometry.
 
#### Resources
- [cdaversin/CerebroSpinalFlow](https://github.com/cdaversin/CerebroSpinalFlow)
- [mri2mesh](https://github.com/scientificcomputing/mri2mesh)
- [MariusCausemann/brain-PVS-SAS-transport](https://github.com/MariusCausemann/brain-PVS-SAS-transport)

### Stage 5: Meshing (Surface $\to$ Volume)
Creating the tetrahedral mesh for FEniCS / FEniCSx.

#### 1. Wildmeshing (Ftetwild)
* **Implementations:**
    * `wildmeshing-python` (Wrapper by [Cecile](https://github.com/cdaversin/wildmeshing-python) or [Marius](https://github.com/MariusCausemann/wildmeshing-python)).
    * [`pytetwild`](https://github.com/pyvista/pytetwild) (PyVista wrapper).
    * [C++ Source](https://github.com/wildmeshing/fTetWild).
* **Pros:** Extremely robust. Can mesh "dirty," self-intersecting, or non-manifold surfaces where other algorithms fail.
* **Cons:** Tricky to install. It is possible to install with [`pip`](https://pypi.org/project/wildmeshing/) on linux x84 and mac arm (but not linux on arm, i.e docker on arm mac). 
* **Note:** They are also working on a [new version](https://github.com/wildmeshing/wildmeshing-toolkit), but this is not ready for us to use.
* **Examples:**
    * Tuturial: https://scientificcomputing.github.io/fenics-in-the-wild
    * https://github.com/MariusCausemann/brain-PVS-SAS-transport

#### 2. SVMTK (Surface Volume Meshing Toolkit)
* **Implementation:** [GitHub Link](https://github.com/SVMTK/SVMTK)
* **Pros:**
    * Wraps CGAL.
    * Specifically designed for biomedical domains.
    * Excellent utilities for surface repair and subdomain marking.
* **Cons:**
    * Only maintained by Lars Magnus Valnes
    * Tricky to install (conda is the best alternative)
* **Examples:**
    * https://github.com/jorgenriseth/gonzo

### Stage 6: Data Mapping (FEniCSx)
Mapping physiological data from NIfTI images to the generated mesh.

* **Workflow:**
    1.  Load NIfTI (via `nibabel`).
    2.  Align coordinate systems (RAS vs LPS).
    3.  Create FEniCS FunctionSpace (e.g., DG0 for voxels, CG1 for continuous fields).
    4.  Probe/Interpolate values.
* **Applications:**
    * Tissue properties (conductivity, elasticity).
    * DTI-based diffusion tensors.
    * Concentration maps.
* **Examples:**
    * https://github.com/jorgenriseth/gonzo

---

## 3. Example Recipe: "The Idealized Pipeline"

A hypothetical workflow to go from raw scan to a Stokes + Advection-Diffusion simulation.

1.  **Preprocessing:**
    * Convert DICOM to NIfTI (`dcm2niix`).
2.  **Segmentation:**
    * Run `mri_synthseg --i input.nii --o seg.nii` (Fast, robust).
3.  **Surface Extraction (SVMTK):**
    * Extract isosurfaces for GM, WM, CSF.
    * Perform boolean unions to close ventricular holes.
    * Preserve sulci depth.
4.  **Meshing (Ftetwild):**
    * Generate tetrahedral mesh from the cleaned surfaces.
    * Mark subdomains (Cell tags) and boundaries (Facet tags).
5.  **Simulation (FEniCSx):**
    * Import XDMF mesh.
    * Define function spaces.
    * Solve Stokes equations (Fluid flow).
    * Solve Advection-Diffusion (Transport).

---

## 4. Demonstration Data

* **[Gonzo](https://zenodo.org/records/14266867):** (GitHub: [jorgenriseth/gonzo](https://github.com/jorgenriseth/gonzo))
* **[OpenNeuro ds004478](https://openneuro.org/datasets/ds004478/versions/1.0.2):** Raw T1w and fMRI BOLD data.
* **[MNI152](https://nist.mni.mcgill.ca/icbm-152-extended-nonlinear-atlases-2020/):** Standard template for registration testing.

---

## 5. Summary of resources

- [MariusCausemann/intracranialPulsation](https://github.com/MariusCausemann/intracranialPulsation) - Paper repo (Legacy FEnICS + multiphenics)
- [MariusCausemann/brain-PVS-SAS-transport](https://github.com/MariusCausemann/brain-PVS-SAS-transport) - Paper repo (Legacy FEnICS)
- [gMRI2FEM](https://github.com/jorgenriseth/gMRI2FEM) - Tool for working with GMRI data (based on FEniCS, SVMTK and many other tools)
- [pantarei](https://github.com/jorgenriseth/pantarei) - A collection of utilities for fluid flow modelling using FEniCS (legacy)
- [Gonzo](https://github.com/jorgenriseth/gonzo) - Full pipline with dataset (based on gMRI2FEM)
- [fenics-in-the-wild](https://github.com/scientificcomputing/fenics-in-the-wild) - Meshing with FtetWil
- [scifem](https://github.com/scientificcomputing/scifem) - Scientific finite element toolbox (also some biomedical) (FEniCSx)
- [mri2mesh](https://github.com/scientificcomputing/mri2mesh) - Tools for converting images to mesh (PyVista + FEniCSx)
- [bzapf/braintransport](https://github.com/bzapf/braintransport) - Paper repo (Legacy FEniCS + dolfin-adjoint)
- [cdaversin/CerebroSpinalFlow](https://github.com/cdaversin/CerebroSpinalFlow) - Full pipeline in FEniCSx
