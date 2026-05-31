# Methodology — Neural Activation Volume Visualization

## Overview

This document describes the technical methodology behind the Neural Activation Volume visualization, covering dataset design, visualization pipeline decisions, and rendering choices.

---

## 1. Dataset Design

### Why synthetic data?

Synthetic data was chosen deliberately for three reasons:

1. **Full reproducibility** — anyone can regenerate the exact dataset from the script, with no external data dependencies
2. **Domain relevance** — the Gaussian cluster structure directly mimics localized activation patterns seen in real fMRI volumetric data and neural network feature maps
3. **Controlled complexity** — parameters (cluster positions, amplitudes, widths) are tunable, enabling iterative visualization refinement

### Gaussian kernel superposition

The activation field is constructed as a weighted sum of seven 3D isotropic/anisotropic Gaussian kernels:

```
A(x,y,z) = Σ aᵢ · exp(−[(x−cxᵢ)²/2σxᵢ² + (y−cyᵢ)²/2σyᵢ² + (z−czᵢ)²/2σzᵢ²])
```

Where:
- `aᵢ` is the amplitude (peak activation) of cluster i
- `(cxᵢ, cyᵢ, czᵢ)` is the cluster center
- `(σxᵢ, σyᵢ, σzᵢ)` controls cluster spread per axis

One dominant central cluster (amplitude 1.0, σ=1.2) represents a primary activation hub, surrounded by six satellite clusters of decreasing amplitude (0.75–0.50) at varied distances — mimicking the hub-and-spoke topology common in neural connectivity models.

### Gradient vector field

The gradient was computed numerically using finite differences (NumPy `np.gradient`):

```python
gx, gy, gz = np.gradient(activation, spacing)
```

This produces a vector field pointing in the direction of steepest activation increase — analogous to the "flow" toward high-activation regions. This is directly visualizable as streamlines, providing a second data dimension beyond the scalar field.

### File format

Data is written as **VTK ImageData XML (.vti)** — ParaView's native rectilinear grid format. This format:
- Requires no external libraries to read in ParaView
- Supports multiple named data arrays in one file
- Is human-readable XML with base64-encoded binary arrays for efficiency

---

## 2. Visualization Pipeline

### Iso-surface (Contour) filter

The Contour filter extracts iso-surfaces at four threshold values: **0.1, 0.3, 0.6, 0.9**.

These thresholds were chosen to reveal hierarchical structure:

| Threshold | Represents |
|---|---|
| 0.1 | Outer boundary — full extent of detectable activation |
| 0.3 | Secondary activation zone — satellite clusters fully visible |
| 0.6 | Primary activation zone — main cluster body |
| 0.9 | Peak core — highest intensity regions only |

Surface **opacity at 0.4** allows all four shells to be simultaneously visible, giving the viewer spatial intuition about the 3D depth structure — a technique borrowed from medical volume visualization.

### Streamline (Stream Tracer) filter

Streamlines are seeded from a **Point Cloud** distribution (100 seed points, radius 6.3, centered at the volume midpoint). Seeds are distributed spherically to ensure coverage across all activation regions.

The **Runge-Kutta 4-5 integrator** was selected for its adaptive step size, which handles the varying gradient magnitudes across the volume more accurately than fixed-step methods.

Streamlines are colored by `ActivationIntensity` at each point along the path, so line color encodes the activation level the streamline passes through — adding a third information dimension to the visualization.

### Color mapping

The **Inferno** colormap was selected for three reasons:
1. **Perceptually uniform** — equal steps in data value produce equal steps in perceived color difference
2. **Monochromatic-safe** — remains interpretable in greyscale and for colorblind viewers
3. **Intuitive directionality** — dark/cool = low activation, bright/warm = high activation matches natural expectations

### Lighting

Phong specular highlights (`Specular = 0.5`) are applied to iso-surfaces to reinforce the perception of 3D curvature. Without specular, transparent overlapping surfaces can appear flat and ambiguous in depth.

---

## 3. Design Decisions & Trade-offs

| Decision | Rationale |
|---|---|
| 64³ grid instead of 128³ | Reduces memory footprint by 8× while preserving visual fidelity for smooth Gaussian fields |
| No noise in v2 | Noise in v1 caused surface fragmentation under the Contour filter; smooth fields produce cleaner iso-surfaces |
| Point Cloud seed over Line Source | Spherical seed distribution better covers a volumetric dataset than a single line |
| 4 iso-surfaces over volume rendering | More GPU-efficient; equally expressive for smooth scalar fields; avoids transfer function tuning complexity |

---

## 4. Reproducibility

The full visualization state is saved in `neural_activation_portfolio.pvsm`. This ParaView state file encodes:
- All filter parameters
- Color map settings and opacity values
- Camera position and orientation
- Lighting configuration

Loading this state file in any ParaView 6.1+ installation (with the `.vti` file in the same directory) will reproduce the exact visualization.
