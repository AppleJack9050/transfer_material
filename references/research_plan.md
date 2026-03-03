# Research Plan (PhD-level, Technical)
---

## 1. Background and Motivation (GIScience Framing)

High-fidelity 3D representations of the built environment are increasingly central to GIScience, spatial analysis, and geospatial decision support systems. Building-scale 3D models enable advanced geographic workflows such as volumetric urban morphology, 3D visibility and shadow modeling, microclimate simulation inputs, and infrastructure monitoring. However, current reconstruction pipelines often fail to meet two operational requirements that are critical in geographic research and practice:

1. **Computational scalability**: the ability to process large extents (neighborhood → city → region) with bounded runtime/memory and without high-end compute infrastructure.
2. **Operational robustness**: the ability to deliver stable, reproducible outputs with minimal manual intervention, parameter tuning, or dataset-specific engineering.

In geographic production settings, the reconstruction system is not judged primarily by rendering realism but by **geospatial integrity**, **analytical utility**, and **workflow reliability**.

---

## 2. Research Aim

The aim of this research is to **design, implement, and systematically evaluate a 3D geospatial reconstruction pipeline** that enables **computationally efficient, scalable, and operationally autonomous processing**, while **preserving the geometric accuracy, precision, and spatial consistency required for GIS-grade downstream analysis** across heterogeneous geographic scenes.

---

## 3. Research Questions

**Primary Research Question (PRQ).**  
How can a 3D geospatial reconstruction **pipeline** be architected to achieve **computational efficiency**, **scalable processing**, and **operational autonomy**, while producing 3D products that are **fit-for-purpose for downstream GIS analysis** across heterogeneous geographic scenes?

---

### RQ1 — Efficiency  
How can reconstruction runtime, memory consumption, and storage requirements be minimized while preserving the geometric fidelity and precision necessary for GIS-grade spatial analysis?

---

### RQ2 — Scalability  
How can the framework scale with increasing mapped extent, data volume, and scene heterogeneity while maintaining stable throughput and consistent geometric quality?

---

### RQ3 — Operational Autonomy  
How can the reconstruction workflow be designed to require minimal manual **parameter specification and hyperparameter tuning**, while maintaining reliable performance across diverse scenes?

---

## 4. Scope and Assumptions

### Inputs
- Primary: multi-view geospatial imagery (aerial/oblique/ground), with metadata or external pose priors when available.
- Optional: depth cues or sparse 3D priors if present (treated as auxiliary constraints, not required).

### Outputs
- Georeferenced building model products suitable for GIS:
  - 3D building mesh or point cloud (primary),
  - optional derived products: height surfaces, footprints with heights, LoD-style simplified models.

### Operational Constraints
- Hardware-friendly deployment targets: single GPU or CPU+GPU setups with bounded memory; avoid assumptions of multi-GPU/HPC.
- Reproducibility: deterministic or near-deterministic pipelines; explicit parameter documentation and automatic logging.

---

## 5. Methodological Design (Technical)

### 5.1 System Architecture Overview
The proposed framework is organized into four coupled subsystems:

1. **Geospatial ingestion and normalization**
2. **Pose estimation and georeferencing**
3. **Compute-efficient building reconstruction**
4. **GIS-ready product generation and QA**

Each subsystem is designed with explicit constraints on computational complexity, error propagation, and operational automation.

---

### 5.2 Geospatial Ingestion and Normalization

**Objectives**
- Normalize imagery radiometry and geometry to reduce dataset-specific brittleness.
- Standardize coordinate reference handling and metadata ingestion.

**Key steps**
- Radiometric normalization: exposure/white-balance stabilization (lightweight, deterministic).
- Image downscaling pyramid for multi-resolution processing.
- CRS management:
  - enforce a declared CRS (EPSG),
  - consistent unit handling (meters),
  - maintain transformation provenance.

**Deliverables**
- A standardized scene package including:
  - imagery pyramid,
  - metadata/priors,
  - spatial extent tiling index,
  - QA-ready logs.

---

### 5.3 Pose Estimation and Georeferencing

**Problem**
Accurate pose estimation is a dominant source of downstream geometric error. For GIS usage, absolute alignment matters (not only relative structure).

**Approach**
- Baseline structure-from-motion for relative pose recovery.
- Georeferencing augmentation:
  - integrate available pose priors (GNSS/IMU, platform telemetry) as soft constraints,
  - incorporate ground control constraints when available,
  - apply robust global bundle adjustment with geospatial regularization.

**Automation mechanisms**
- Automatic detection of ill-conditioned geometry (low parallax, poor overlap).
- Quality gating on:
  - reprojection error distribution,
  - pose graph connectivity,
  - local drift indicators.

**Output**
- Georeferenced camera trajectory/poses with uncertainty estimates where feasible.

---

### 5.4 Compute-Efficient Building Reconstruction

**Design principle**
Efficiency is achieved by restricting computation to the spatial and visual support needed for building geometry, using compact representations and bounded-memory processing.

#### 5.4.1 Spatial Tiling and Bounded-Memory Mapping
- Partition the mapped extent into spatial tiles (e.g., fixed meter-grid or adaptive quadtree).
- Process tiles independently with overlap margins to manage boundary artifacts.
- Maintain a global index for:
  - shared features,
  - cross-tile alignment constraints,
  - deduplication during fusion.

**Benefits**
- Memory bounded by tile size (not total area).
- Parallelizable and scalable to large extents.

#### 5.4.2 Adaptive Resolution Control
- Use multi-resolution reconstruction:
  - coarse pass for global structure,
  - refinement pass only where geometric complexity warrants it.
- Control refinement with structural criteria (e.g., surface residuals, planarity deviations).

#### 5.4.3 Lightweight Model Representation
Candidate representations (final choice depends on experimental outcomes):
- Compact point-based or surfel representation with pruning,
- TSDF-style fusion (if depth is available or can be estimated reliably),
- Sparse implicit field with efficient encoding and aggressive early termination.

**Compute optimizations**
- Early discard of non-informative pixels/regions via building likelihood (from priors or lightweight cues).
- Pruning of redundant primitives (points/surfels/gaussians) based on contribution.
- Quantization:
  - FP16 for compute stages,
  - optional INT8 for stored parameters where safe.

---

### 5.5 Workflow Automation and User-Friendliness (Operational Robustness)

**Goal**
Minimize manual operation by replacing expert choices with data-driven defaults and explicit failure detection.

**Key components**
- **Auto-configuration layer**:
  - selects parameters based on measurable dataset properties:
    - overlap statistics,
    - resolution,
    - expected scale (GSD),
    - feature density.
- **Quality assurance gates**:
  - triggers reconfiguration or fallback modes when metrics exceed thresholds:
    - pose instability,
    - excessive residuals,
    - incomplete coverage indicators.
- **Reproducibility tooling**:
  - configuration snapshots,
  - deterministic seeds,
  - full provenance logs (inputs → outputs).

**User interaction model**
- Single entry-point configuration (minimal knobs).
- Standardized outputs + QA report.

---

### 5.6 GIS-Ready Output Products and Spatial Validity

**Primary products**
- Georeferenced point cloud or mesh with CRS annotation and metric units.

**Derived GIS products**
- Building height surfaces (DSM/DTM-derived if applicable),
- footprint + height attributes (vectorized),
- LoD-style simplified building solids for analysis.

**Validation of spatial validity**
- Confirm that derived measures (heights, volumes, orientations) remain stable under reasonable perturbations of the pipeline parameters.

---

## 6. Evaluation Framework (Technical and Geographic)

### 6.1 Geometry Fidelity (Metric)
- Point-to-surface distance / cloud-to-mesh distance against reference if available.
- Structural diagnostics:
  - planarity residuals,
  - vertical consistency,
  - edge sharpness measures.

### 6.2 Geospatial Integrity
- Absolute positional error (horizontal and vertical) where reference control exists.
- CRS correctness and transformation audit:
  - reproducible transforms,
  - unit correctness,
  - provenance completeness.

### 6.3 Scalability and Efficiency
Report at minimum:
- runtime per km²,
- peak memory per km²,
- output size per km²,
- throughput under constrained hardware profiles.

### 6.4 Operational Robustness and Usability
- Operator time (manual steps count and duration).
- Parameter sensitivity analysis:
  - variance of outputs under small parameter changes,
  - stability across heterogeneous scenes.
- Failure mode taxonomy with automatic detection performance.

### 6.5 Baselines for Comparison
- Classical: SfM + MVS pipeline with standard settings.
- Production-style: geospatial fusion pipeline (if applicable).
- Any additional baseline chosen for relevance to the target input data.

---

## 7. Expected Contributions (PhD-level)

1. **A bounded-memory, tile-based geospatial reconstruction pipeline** enabling large-area processing without HPC assumptions.
2. **A compute-efficient reconstruction strategy** (representation + pruning/quantization/adaptive refinement) with measured gains in runtime and memory.
3. **An operational automation layer** that reduces manual tuning, improves reproducibility, and provides QA diagnostics suitable for geographic data production.
4. **A geospatial integrity protocol** (CRS handling + provenance + spatial validity checks) to ensure outputs are fit for GIS-based analysis.

---

## 8. Limitations and Risk Management

- Georeferencing quality is bounded by input priors and scene geometry; mitigation includes robust constraints and QA-triggered fallback modes.
- Scene heterogeneity (lighting, viewpoints, occlusion) can destabilize pose estimation; mitigation includes multi-resolution processing, robust matching, and auto-reconfiguration.
- Efficiency–accuracy trade-offs will be explicitly characterized through Pareto-front evaluation rather than a single operating point.

---