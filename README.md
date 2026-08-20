SSM_R_AAOCA — Statistical Shape Modeling pipeline for R-AAOCA
PCA-based SSM of the aortic root and intramural right coronary artery. This repository contains
the full processing pipeline used to go from segmented CTA volumes to the
final principal-component shape modes and their correlation with clinical
variables.
Pipeline overview and mapping to the paper's Methods section
#	Script	What it does
1	`scripts/01\\\_aorta\\\_preprocessing.py`	Extracts the aorta/LV segmentation masks from the `.nrrd` volume, cleans the mask, computes a centerline, resamples at 0.5 mm and smooths with a cubic B-spline.
2	`scripts/02\\\_rca\\\_preprocessing.py`	Isolates the two candidate coronary components, identifies the ostium, prunes secondary branches from the skeleton, and computes the RCA centerline from the ostium to the most distal point.
3	`scripts/03\\\_aorta\\\_rigid\\\_alignment.py`	Rigid (Kabsch) alignment of every aorta centerline/mask to a common template (the case closest to the median anatomy).
4	`scripts/04\\\_rca\\\_rigid\\\_alignment.py`	Weighted Kabsch alignment of the RCA, with the ostium heavily weighted, to improve ostial consistency across subjects.
5	`scripts/05\\\_length\\\_normalization.py`	Clips every centerline/mask to a fixed cut-length (robust percentile of the cohort) so all subjects share a comparable extent.
6	`scripts/06\\\_masktomesh.py`	Converts the clipped binary masks into triangulated surface meshes via marching cubes, with a multi-step watertight repair (hole capping, non-manifold repair, Taubin smoothing).
7	`scripts/07\\\_remesh.py`	Uniform mesh resampling to a fixed vertex count (aorta: 5,000; RCA: 2,000) using the ACVD algorithm, with robust normal-orientation fixing.
8	`scripts/08\\\_registration.py`	Point-to-point correspondence via rigid pre-alignment + deformable Coherent Point Drift (CPD) registration of every subject mesh to the template.
9	`scripts/09\\\_pca.py`	Fits PCA on the registered shape vectors (aorta and RCA modeled separately), selects the components explaining up to 90% cumulative variance, and computes Spearman correlations between shape scores and clinical variables (FDR-corrected).

#Data
Patient CTA imaging data and derived segmentations are not included in
this repository due to institutional and ethics-committee restrictions on
patient data.
The scripts expect the following input layout, which you can recreate with
your own (locally stored) segmented `.nrrd` volumes:
```
data/00\_raw\_nrrd/<patient\_id>.nrrd
```
Each `.nrrd` file is expected to contain 3D-Slicer-style segment labels for
the aortic root (`Segment\_1`), the coronary arteries (`Segment\_2`), and the
left ventricle (`Segment\_3`)

#Setup
```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\\Scripts\\activate
pip install -r requirements.txt
```
All pipeline directories are defined in a single place, `config.py`.
By default everything reads/writes under `./data`; to point the pipeline at
a different location (e.g. an external drive), set the `SSM\_DATA\_DIR`
environment variable before running any script:
```bash
export SSM\_DATA\_DIR=/path/to/your/dataset
python config.py   # creates the full folder tree under $SSM\_DATA\_DIR
```
Running the pipeline
Scripts are meant to be run in numerical order, each one consuming the
output of the previous step:
```bash
python scripts/01\_aorta\_preprocessing.py
python scripts/02\_rca\_preprocessing.py
python scripts/03\_aorta\_rigid\_alignment.py
python scripts/04\_rca\_rigid\_alignment.py
python scripts/05\_length\_normalization.py
python scripts/06\_masktomesh.py
python scripts/07\_remesh.py
python scripts/08\_registration.py
python scripts/09\_pca.py
```
Step 9 additionally expects a clinical spreadsheet at
`data/clinical/clinical\_data.xlsx` with an `ID` column matching the patient
IDs used throughout the pipeline, plus the clinical variables listed in
`CLINICAL\_VARS` at the top of `09\_pca.py`.
