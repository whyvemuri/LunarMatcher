[README.md](https://github.com/user-attachments/files/31969712/README.md)
# LunarMatcher — Chandrayaan-2 Multimodal Image Correspondence

<p align="center">
  <b>Multi-modal, sun-angle and scale-invariant image correspondence for Chandrayaan-2 optical imagery</b>
</p>

<p align="center">
  OHRC · TMC · IIRS · PDS4 · Photometric Normalization · SIFT/ORB · RANSAC
</p>

---

## 🌙 Overview

**LunarMatcher** is a multimodal lunar-image registration pipeline designed to find reliable correspondences between optical images acquired by the **Chandrayaan-2 Orbiter** and register images from different sensors despite differences in illumination, viewpoint, scale, and spatial resolution.

The project targets the problem of registering Chandrayaan-2 **OHRC, TMC, and IIRS** observations. The underlying problem statement calls for a generic correspondence solution capable of producing distributed match points and sub-pixel-accurate registration, with evaluation using metrics such as RMSE, inlier count, and inlier ratio.

### Why this is difficult

Lunar images captured at different times and by different instruments can vary significantly because of:

- ☀️ **Sun-angle / illumination variation**
- 📐 **Viewpoint and geometric distortion**
- 🔍 **Scale and spatial-resolution variation**
- 🛰️ **Different sensor modalities**
- 🌑 Strong lunar-regolith photometric effects
- 💾 Very large raw PDS4 image products

The pipeline therefore combines metadata-aware preprocessing, photometric normalization, geographic overlap filtering, feature-based registration, and memory-conscious I/O.

---

## 🚀 Live Project

**Interactive web interface:**  
https://lunarmatcherv3.netlify.app/#pipeline

The web interface presents the LunarMatcher pipeline and its workflow.

---

## 🎯 Project Objectives

The pipeline is designed to:

1. Load Chandrayaan-2 OHRC, TMC, and IIRS products.
2. Read image geometry and acquisition metadata from PDS4 XML labels.
3. Determine geographic overlap before attempting registration.
4. Crop overlapping regions instead of processing complete products.
5. Normalize image intensity and compensate for illumination differences.
6. Generate a compact panchromatic representation of hyperspectral IIRS data.
7. Detect and match scale-space features.
8. Estimate geometric transformation using RANSAC.
9. Warp the source image into the reference-image coordinate system.
10. Quantitatively evaluate registration quality.
11. Keep memory usage manageable for multi-gigabyte planetary datasets.

---

## 🛰️ Supported Chandrayaan-2 Sensors

| Sensor | Data role | Pipeline handling |
|---|---|---|
| **OHRC** | High-resolution optical imagery | PDS4 image loading + overlap cropping + preprocessing |
| **TMC** | Terrain Mapping Camera imagery | PDS4 image loading + overlap cropping + preprocessing |
| **IIRS** | Imaging Infrared Spectrometer | Chunked cube loading + PCA panchromatic proxy |

The implemented pairwise workflows are:

- **OHRC ↔ TMC**
- **OHRC ↔ IIRS**
- **TMC ↔ IIRS**

---

## 🧩 Pipeline

```text
                    Chandrayaan-2 Products
                             │
              ┌──────────────┼──────────────┐
              │              │              │
             OHRC           TMC            IIRS
              │              │              │
              └──────────────┼──────────────┘
                             │
                     PDS4 XML Metadata
                             │
                  Geometry / Dimensions /
                  Data Type / Sun Angles
                             │
                             ▼
                 Geographic Overlap Check
                             │
                             ▼
                    Overlapping Crop
                             │
                             ▼
                 Memory-Safe Image Loading
                             │
                             ▼
              Photometric / Sun-Angle Correction
                             │
                  ┌──────────┴──────────┐
                  │                     │
          OHRC / TMC preprocessing    IIRS PCA
                  │                     │
                  └──────────┬──────────┘
                             ▼
                     Scale-Space Features
                         SIFT / ORB
                             │
                             ▼
                       BF Matching
                   + Lowe Ratio Test
                             │
                             ▼
                   RANSAC Homography
                             │
                             ▼
                     Image Registration
                             │
                             ▼
                 Validation & Diagnostics
          ┌──────────┬─────────┬─────────┬─────────┐
          │          │         │         │         │
        Matches   Inliers    RMSE      SSIM       MI
                             │
                             ▼
                     Registered Product
```

---

## 🔬 Core Methodology

### 1. PDS4 metadata parsing

The pipeline reads the accompanying XML labels rather than assuming image dimensions or data types.

Metadata extraction includes:

- Image dimensions
- Pixel data type
- Geographic corner coordinates
- Incidence angle
- Emission angle
- Phase angle
- Acquisition time

This allows the pipeline to adapt to different Chandrayaan-2 products.

---

### 2. Memory-safe image loading

The raw products are extremely large, so loading complete images into RAM is avoided wherever possible.

The implementation uses:

- `numpy.memmap`
- Downsampling
- Cropping by geographic overlap
- Explicit cache management
- Garbage collection
- RAM checkpoints
- Chunked IIRS processing

The PDS4 image loader reads the data through memory mapping and reduces the working image to a configurable maximum dimension.

---

### 3. Geographic overlap filtering

Before feature matching, the pipeline computes bounding-box overlap using the geographic corner coordinates extracted from the PDS4 labels.

This prevents wasting computation on image pairs that do not observe the same lunar region.

Conceptually:

```text
Image A footprint
┌─────────────────────┐
│                     │
│       ┌─────────────┼──────┐
│       │   overlap   │      │
└───────┼─────────────┘      │
        │                    │
        └────────────────────┘
              Image B
```

Only overlapping regions proceed to registration.

---

## ☀️ Photometric / Sun-Angle Normalization

The pipeline provides three interchangeable photometric models:

### Cosine correction

A simple Lambertian-style baseline:

```text
I_corrected ∝ cos(i_ref) / cos(i)
```

where `i` is the observed incidence angle and `i_ref` is the chosen reference incidence angle.

### Lommel-Seeliger correction

The default implementation uses a Lommel-Seeliger model, which is better suited to rough, particulate planetary surfaces than a simple Lambertian correction at large illumination angles.

### Simplified Hapke correction

A simplified Hapke-style model is also implemented using lunar-regolith parameters:

- Single-scattering albedo: `w = 0.21`
- HG asymmetry parameter: `g = -0.4`

The implementation explicitly simplifies the full Hapke radiative-transfer model by omitting the opposition-surge and multiple-scattering corrections.

### Contrast enhancement

After photometric normalization, CLAHE is applied to improve local contrast and make feature detection more robust.

---

## 🗻 Topographic Correction

A topographic-correction function is included, but it is **not enabled by default**.

The current dataset used by the notebook contains raw OHRC/TMC/IIRS strips but does not include a DEM required for per-pixel surface-normal correction.

Once a suitable DEM is available, the implemented function can use:

- DEM slope
- DEM aspect
- Solar incidence angle
- Solar azimuth

to compensate for local topographic illumination.

```text
DEM
 │
 ├── Slope
 └── Aspect
      │
      ▼
Local surface normal
      │
      ▼
Local illumination correction
```

---

## 🌈 IIRS Processing

IIRS data is hyperspectral and significantly larger than the 2D OHRC/TMC images.

Instead of loading the complete cube into memory, the pipeline:

1. Reads the PDS4 dimensions from the XML label.
2. Memory-maps the raw `.qub` file.
3. Processes spectral bands in chunks.
4. Downsamples the long line dimension.
5. Builds a reduced hyperspectral cube.
6. Applies `IncrementalPCA`.
7. Uses the first principal component as a **panchromatic proxy**.
8. Reuses the generated proxy through a cache.

This reduces memory pressure while retaining a compact representation for image correspondence.

Example proxy size observed in the notebook:

```text
~2000 × 250 × 256
≈ 512 MB
```

---

## 🔎 Feature Detection and Matching

The registration stage currently supports:

### SIFT

```python
cv2.SIFT_create()
```

SIFT descriptors are matched using a brute-force matcher with the L2 distance.

### ORB

ORB is also supported:

```python
cv2.ORB_create(nfeatures=4000)
```

ORB descriptors use Hamming distance.

### Ratio test

Candidate matches are filtered using a nearest-neighbor ratio test.

Default:

```text
ratio_thresh = 0.75
```

---

## 📐 Geometric Registration

After feature matching, the pipeline estimates a projective transformation using a **homography**.

```python
H, mask = cv2.findHomography(
    ptsB,
    ptsA,
    cv2.RANSAC,
    ransac_thresh
)
```

Default RANSAC threshold:

```text
5.0 pixels
```

The resulting homography is then used to warp the moving image onto the reference image.

---

## 📊 Evaluation Metrics

The pipeline reports several complementary metrics.

| Metric | Purpose |
|---|---|
| **Number of matches** | Number of feature correspondences surviving the ratio test |
| **Inlier count** | Matches consistent with the estimated homography |
| **Inlier ratio** | Fraction of matches accepted by RANSAC |
| **RMSE** | Pixel-level residual error between overlapping registered regions |
| **SSIM** | Structural similarity after registration |
| **Mutual Information** | Statistical dependence between registered image intensities |
| **Overlap %** | Percentage of the reference image participating in the comparison |

The notebook also visualizes:

- Feature matches
- Warped image
- Residual/error map
- Tie-point displacement vectors

---

## 📈 Example Run

The included notebook contains completed sample runs for OHRC/IIRS and TMC/IIRS registration.

Some representative observed results include:

| Pair | Matches | Inliers | Inlier ratio | RMSE | SSIM | MI | Overlap |
|---|---:|---:|---:|---:|---:|---:|---:|
| OHRC ↔ IIRS | 8 | 4 | 50.0% | 85.43 | 0.061 | 0.013 | 22.6% |
| OHRC ↔ IIRS | 7 | 5 | 71.4% | 94.31 | 0.064 | 0.012 | 36.5% |
| TMC ↔ IIRS | 23 | 6 | 26.1% | 113.15 | 0.221 | 0.073 | 76.6% |
| TMC ↔ IIRS | 39 | 6 | 15.4% | 68.72 | 0.179 | 0.045 | 70.4% |

These are **sample notebook outputs, not a claim of final system performance**. The relatively low inlier counts and variable similarity scores also show that cross-modal lunar registration remains challenging and requires further refinement.

---

## 💾 Memory Optimization

Large planetary image products can quickly exceed Colab RAM.

The pipeline therefore includes:

- `numpy.memmap`
- Geographic cropping before matching
- Downsampling
- Chunked IIRS loading
- Incremental PCA
- Explicit cache clearing
- `gc.collect()`
- RAM usage logging
- Reuse of IIRS PCA products
- Pairwise execution to prevent a single failed group from terminating the complete workflow

RAM usage is periodically written to:

```text
ram_debug_log.txt
```

---

## 🛠️ Tech Stack

### Programming

- Python 3
- Google Colab

### Scientific / Image Processing

- NumPy
- OpenCV
- scikit-image
- scikit-learn
- Matplotlib

### Data Handling

- PDS4 XML labels
- `.img` image products
- IIRS `.qub` products
- NumPy memory mapping

### Deployment / Interface

- Netlify
- Web-based LunarMatcher interface

---

## 📦 Installation

The notebook installs its primary Python dependencies with:

```bash
pip install -q psutil opencv-python-headless scikit-image scikit-learn
```

Additional imports used by the pipeline include:

```text
numpy
matplotlib
xml.etree.ElementTree
gc
os
glob
```

---

## ▶️ Running the Pipeline

### 1. Open the notebook

Open:

```text
CH2_pipeline_v3 (1).ipynb
```

in Google Colab.

### 2. Mount Google Drive

The notebook expects the Chandrayaan-2 data directory at:

```text
/content/drive/MyDrive/sih_chandrayaan2
```

Update the `FOLDER` variable if your dataset is stored elsewhere.

### 3. Place the data

The pipeline expects each image product to be accompanied by its metadata label.

Typical structure:

```text
sih_chandrayaan2/
├── ch2_ohr_*.img
├── ch2_ohr_*.xml
├── ch2_tmc_*.img
├── ch2_tmc_*.xml
├── ch2_iir_*.qub
├── ch2_iir_*.xml
└── ram_debug_log.txt
```

### 4. Run cells sequentially

The notebook is organized into numbered stages:

```text
0. Install dependencies
1. Mount Drive / imports / RAM logging
2. Define input files and integrity checks
3. Parse geometry and acquisition metadata
4. Memory-safe PDS4 loading
5. Photometric normalization
6. Topographic correction stub
7. SIFT/ORB registration + metrics
8. IIRS PCA panchromatic proxy
9. OHRC ↔ TMC
10. OHRC ↔ IIRS
11. TMC ↔ IIRS
12. Optional RAM log inspection
```

---

## 🗂️ Recommended Repository Structure

```text
LunarMatcher/
│
├── README.md
│
├── notebooks/
│   └── CH2_pipeline_v3.ipynb
│
├── src/
│   ├── io.py
│   ├── metadata.py
│   ├── photometric.py
│   ├── iirs.py
│   ├── registration.py
│   └── metrics.py
│
├── data/
│   ├── README.md
│   └── .gitkeep
│
├── results/
│   ├── matches/
│   ├── registered/
│   └── metrics/
│
├── docs/
│
└── requirements.txt
```

> The raw Chandrayaan-2 datasets should **not** be committed to GitHub. Keep large mission products in the appropriate data repository/storage location and document how to obtain them.

---

## 🛰️ Dataset Sources

The project is based on Chandrayaan-2 optical products including:

- OHRC
- TMC-2
- IIRS

Reference imagery can include:

- Lunar Reconnaissance Orbiter (LRO) NAC
- SELENE imagery

The problem statement identifies the Chandrayaan-2 data portal and LRO resources as dataset sources.

### Chandrayaan-2 data

https://chmapbrowse.issdc.gov.in/

### LRO NAC

https://lroc.im-ldi.com/images/downloads/

### LRO QuickMap

https://quickmap.lroc.im-ldi.com/

---

## ⚠️ Current Limitations

This version should be considered a **research/prototype pipeline**, not a finished production registration system.

### 1. Cross-modal matching remains difficult

OHRC, TMC, and IIRS observe lunar terrain differently. A feature that is visually distinctive in one modality may not have the same appearance in another.

### 2. IIRS sun-angle metadata

The notebook notes that IIRS products may not contain native sun-angle information. In such cases, the pipeline estimates an incidence angle using temporally related reference observations.

This should be treated as an approximation.

### 3. Topographic correction is currently disabled

A suitable DEM is required before the topographic correction stage can be used.

### 4. IIRS PCA is a proxy

The current IIRS representation uses the first principal component as a panchromatic proxy rather than a complete physically motivated multispectral/hyperspectral registration representation.

### 5. Homography is an approximation

A homography is useful for planar/projective alignment, but lunar terrain is inherently 3D. Large terrain relief and viewing-angle changes can violate the homography assumption.

### 6. Sub-pixel accuracy is not yet demonstrated

The original problem statement expects sub-pixel correspondence accuracy. The current notebook reports pixel-domain registration metrics, but the available sample outputs should not be interpreted as proof of sub-pixel accuracy.

---

## 🔮 Future Work

Potential improvements include:

- [ ] Add DEM-based topographic correction
- [ ] Improve IIRS physical preprocessing
- [ ] Use robust multimodal descriptors
- [ ] Add phase-correlation / intensity-based refinement
- [ ] Add dense correspondence after sparse matching
- [ ] Perform sub-pixel tie-point refinement
- [ ] Add adaptive feature selection
- [ ] Improve spatially uniform tie-point distribution
- [ ] Add confidence scoring for individual matches
- [ ] Compare SIFT, ORB and modern learned feature methods
- [ ] Add automated batch processing
- [ ] Export registered products and match-point files
- [ ] Add reproducible experiment configuration
- [ ] Build automated benchmark datasets
- [ ] Validate against known ground-control points

---

## 🧪 Reproducibility

For reproducible experiments, record:

```text
Dataset/product IDs
Sensor pair
Photometric model
Reference incidence angle
Feature detector
Ratio-test threshold
RANSAC threshold
Downsampling parameters
IIRS PCA parameters
Registration metrics
```

A recommended experiment record is:

```text
Experiment
├── input products
├── preprocessing configuration
├── matching configuration
├── homography
├── match points
├── inlier mask
├── registered image
└── evaluation metrics
```

---

## 👥 Project

**Problem Statement ID:** 26166

**Problem Statement:**  
*Multi-modal, Sun angle and scale invariant image correspondence using Chandrayaan-2 optical images (OHRC, TMC and IIRS)*

**Organization:**  
Indian Space Research Organisation (ISRO)

**Department:**  
Department of Space / Indian Space Research Organisation

**Category:**  
Software

**Theme:**  
Space Technology

---

## 📚 References & Data

The implementation is based on the project problem statement and the accompanying Chandrayaan-2 datasets and metadata.

For the scientific models, consult the relevant literature for:

- Lunar photometry
- Lommel-Seeliger reflectance
- Hapke reflectance theory
- SIFT feature detection
- RANSAC model estimation
- PDS4 planetary data standards
- Chandrayaan-2 OHRC/TMC/IIRS instrument products

---

## 📄 License

Add the project's chosen license here before publishing the repository.

For example:

```text
MIT License
```

Do not assume a license for mission data or third-party datasets. Their respective data-use conditions apply.

---

<p align="center">
  Built for multimodal lunar image correspondence 🚀🌙
</p>
