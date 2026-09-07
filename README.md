## Thermal Modeling and Numerical Validation of a PCB Plate in TVAC Environment

> **A comparative study: Experimental data vs. Python nodal network vs. Simcenter 3D**

This repository contains the full numerical pipeline, experimental data processing, MDAO parameter identification, and comparative validation of transient 2D thermal models for an FR4 printed circuit board (PCB) tested inside a thermal vacuum chamber (TVAC).

---

## Table of Contents

* [Overview]
* [Physical Problem]
* [Repository Structure]
* [Data Files]
* [Numerical Methods]
* [MDAO Parameter Identification]
* [Validation Pipeline]
* [Key Results]
* [Installation and Requirements]
* [Usage]
* [Figures and Plots]
* [References]
* [License]

---

## Overview

This project validates transient thermal models of a square FR4 PCB plate heated by a centralized electrical heater inside a cryogenic thermal vacuum chamber (TVAC). Three independent sources are cross-validated:

| Source | Type | Description |
| --- | --- | --- |
| **Experimental** | TVAC telemetry | Type-K thermocouple measurements at PCB center & corner nodes |
| **Python 2D** | Nodal network | Custom finite-difference implicit solver with nonlinear radiation |
| **Simcenter 3D** | 3D FEM | Siemens commercial finite element reference model |

All numerical results are synchronized with experimental data via discrete cross-correlation, and point-by-point relative errors are computed to quantify model fidelity.

---

## Physical Problem

### Geometry & Material

| Parameter | Value |
| --- | --- |
| Plate dimensions | 100 mm $\times$ 100 mm $\times$ 1.6 mm |
| Material | Pure FR4 (isotropic) |
| Heater area | 80 mm $\times$ 80 mm (64% of bottom face, centered) |
| Heater efficiency | $\eta = 0.6$ (40% electrical/parasitic losses) |

### Thermophysical Properties (nominal baseline)

| Parameter | Symbol | Range | Nominal |
| --- | --- | --- | --- |
| Density | $\rho$ | $[1800, 2000]\text{ kg/m}^3$ | 1900 |
| Thermal conductivity | $k$ | $[0.25, 0.8]\text{ W/(m}\cdot\text{K)}$ | 0.5 |
| Specific heat | $c_p$ | $[600, 1200]\text{ J/(kg}\cdot\text{K)}$ | 900 |
| Emissivity | $\varepsilon$ | $[0.7, 0.95]$ | 0.9 |

### Boundary Conditions

* **Top surface**: Radiative heat loss to cryogenic shroud (Stefan-Boltzmann, $\varepsilon \approx 0.9$)
* **Bottom surface (outside heater)**: Adiabatic (insulated)
* **Four lateral edges**: Adiabatic (insulated)
* **Heater input**: Stepped power profile ($2\text{ W} \rightarrow 10\text{ W} \rightarrow 5\text{ W}$) with 40% losses
* **Shroud temperature**: $\sim 70\text{ K}$ (cryogenic $\text{LN}_2$), with experimental dynamic telemetry option
* **Initial temperature**: $\sim 200\text{ K}$ (uniform)

### Power Profile

| Time | $P_{\text{elec}}$ | Loss | $P_{\text{eff}}$ |
| --- | --- | --- | --- |
| $[0, 10000]\text{ s}$ | 2.0 W | 40% | 1.2 W |
| $]10000, 17500]\text{ s}$ | 10.0 W | 40% | 6.0 W |
| $]17500, 20000]\text{ s}$ | 5.0 W | 40% | 3.0 W |

### TVAC Facility

Experiments conducted at the **Center for Nanosatellite Testing (CeNT)**, Kyushu Institute of Technology (Kyutech). Chamber: $\Phi 310\text{ mm} \times \text{L} 400\text{ mm}$, internal shroud $\Phi 245\text{ mm} \times \text{L} 360\text{ mm}$, high vacuum ($< 10^{-3}\text{ Pa}$), $\text{LN}_2$ cooling to $\sim 70\text{ K}$.

---

## Repository Structure

```text
.
├── Plate Analysis.ipynb          # Main Jupyter Notebook (full pipeline)
├── data/
│   ├── PCB_Rad_Summary.xlsx      # Experimental TVAC telemetry (thermocouples, shroud, power)
│   ├── Power_Green_White.xlsx     # Raw electrical power supply data
│   └── SimcenterData.xlsx         # Simcenter 3D FEM exported results (Central & Corner sheets)
├── figures/                       # Generated plots and figures (optional, auto-saved)
├── README.md                      # This file
└── requirements.txt               # Python dependencies

```

---

## Data Files

### `data/PCB_Rad_Summary.xlsx`

Primary experimental dataset. Sheet: **"Green and White 1"**.

| Column | Description |
| --- | --- |
| `Time` | Absolute timestamp (datetime) |
| `Green Center` | Thermocouple temperature at PCB geometric center [°C] |
| `Green Corner (back)` | Thermocouple temperature at PCB corner (unheated zone) [°C] |
| `Shroud Top` | Top shroud wall temperature [°C] |
| `Shroud Bottom` | Bottom shroud wall temperature [°C] |
| `Shroud LN2 Control` | Cryogenic control system telemetry |

**Test window**: 23 April 2026, 10:00 $\rightarrow$ 16:00 (normalized to $t = 0 \rightarrow 20000\text{ s}$).

### `data/Power_Green_White.xlsx`

Raw DC power supply logging (GW Instek PSW80-13.5). Contains voltage/current timestamps used to reconstruct the continuous experimental power profile $P(t)$.

### `data/SimcenterData.xlsx`

Simcenter 3D FEM exported nodal temperature results.

| Sheet | Content |
| --- | --- |
| `Central` | Time + temperature at central node [°C] |
| `Corner` | Time + temperature at corner node [°C] |

Both sheets are filtered to $t \le 19800\text{ s}$ to match the experimental window.

---

## Numerical Methods

### 1. Explicit Forward Euler Scheme

* 2D finite-difference nodal network (5-point stencil)
* Conduction + radiation + heater source evaluated at time step $t^n$
* No matrix inversion required
* **Stability constraint (CFL/Fourier)**:
* 2D conductive limit: $\text{Fo} \le 1/4 \rightarrow \Delta t_{\text{max}} = \frac{\rho \cdot c_p \cdot \Delta x^2}{4k}$
* Radiative limit: $\Delta t_{\text{max}} = \frac{\rho \cdot c_p \cdot e}{4 \cdot \varepsilon \cdot \sigma \cdot T^3}$
* Coupled limit: $\Delta t_{\text{max}} = \frac{\rho \cdot c_p \cdot e \cdot \Delta x^2}{4 \cdot k \cdot e + h_{\text{rad}} \cdot \Delta x^2}$



> **Example**: At $\Delta x = 2\text{ mm}$, $\Delta t_{\text{max}} \approx 3.24\text{ s}$ (conduction-dominated). The explicit scheme diverges explosively if violated.

### 2. Hybrid Implicit Scheme (Production Solver)

* **Conduction**: Fully implicit at $t^{n+1}$ (bypasses CFL constraint)
* **Radiation**: First-order Taylor linearization of $T^4$ around $T^n$ ($h_{\text{rad}} = 4\varepsilon\sigma(T^n)^3$)
* Solved via sparse direct solver (`scipy.sparse.linalg.spsolve`)
* **Unconditionally stable** (strictly diagonally dominant matrix $A^n$ for any $\Delta t > 0$)

### Mesh Convergence Study

Grid independence verified across 7 resolutions ($10 \times 10 \rightarrow 200 \times 200$) with $\Delta t = 200\text{ s}$:

| Mesh | $\Delta x$ [mm] | Mean rel. err. center | Mean rel. err. corner |
| --- | --- | --- | --- |
| $10 \times 10$ | 10.0 | 0.125% | 0.124% |
| $25 \times 25$ | 4.0 | 1.760% | 2.005% |
| $50 \times 50$ | 2.0 | 0.005% | 0.006% |
| $100 \times 100$ | 1.0 | 0.001% | 0.001% |
| $200 \times 200$ | 0.5 | 0.000% | 0.000% |

**Selected production mesh**: $50 \times 50$ (optimal accuracy/cost trade-off).

### Boundary Input Sensitivity (4 Cases)

| Case | Shroud BC | Power input |
| --- | --- | --- |
| 1 | Dynamic $T(t)$ | Real $P(t)$ |
| 2 | Dynamic $T(t)$ | Stepped $P_{\text{step}}$ |
| 3 | Constant 70 K | Real $P(t)$ |
| 4 | Constant 70 K | Stepped $P_{\text{step}}$ |

**Key finding**: Shroud temperature approximation causes the largest error ($+3.02\text{ °C}$ RMSE at corner). Power step discretization is negligible ($+0.22\text{ °C}$).

---

## MDAO Parameter Identification (CasADi + IPOPT)

Material properties ($c_p, k, \varepsilon$) carry intrinsic uncertainties. To isolate the **pure numerical model error** from input parameter noise, an optimization framework identifies the optimal parameter vector $\theta^* = [c_p^*, k^*, \varepsilon^*]$ that minimizes the sum of squared residuals against experimental telemetry.

### Method

1. **Symbolic model** built in CasADi (MX expressions) — a coarse $20 \times 20$ explicit model with collocation integrator for unconditionally stable time-stepping
2. **Objective function**: SSE between simulated and experimental center + corner temperatures
3. **Optimizer**: IPOPT (interior-point) with exact algorithmic differentiation (gradients + Hessians computed by CasADi)
4. **Bounds**: Physical property ranges from Table above
5. **Result**: Optimal parameters injected back into the full $50 \times 50$ implicit production solver

### Optimized Parameters

| Parameter | Value | Unit |
| --- | --- | --- |
| $c_p$ | 842.32 | $\text{J/(kg}\cdot\text{K)}$ |
| $k$ | 0.800 | $\text{W/(m}\cdot\text{K)}$ |
| $\varepsilon$ | 0.931 | — |

### Interpretation

Once $\theta^*$ is found, any remaining residual error is attributable strictly to:

* Spatial discretization truncation ($\Delta x, \Delta y$)
* 2D planar assumption (neglected out-of-plane gradients)
* Unmodeled boundary dynamics (3D edge radiation, parasitic conduction)

---

## Validation Pipeline

The notebook follows a unified data processing workflow for all three sources:

```text
Raw data (Excel) ➔ Temporal windowing ➔ Time normalization (t₀ = 0) 
➔ Cross-correlation alignment ➔ Curve superposition ➔ Relative error computation

```

### Cross-Correlation Alignment

Optimal time lag $k$ is found by maximizing:

$$R_{\text{exp,num}}[k] = \sum (T_{\text{exp}}[n] - \bar{T}_{\text{exp}}) \cdot (T_{\text{num}}[n+k] - \bar{T}_{\text{num}})$$

This eliminates phase lag artifacts between experimental and numerical transient responses.

### Relative Error Metric

$$\text{err}_{\text{rel}}(t) = 100 \times \frac{\vert{}T_{\text{sim}}(t) - T_{\text{exp}}(t)\vert{}}{\vert{}T_{\text{exp}}(t)\vert{} + 273.15}$$

Computed on a common 5000-point interpolated time grid for all pairwise comparisons.

---

## Key Results

### Mean Relative Errors (MDAO-optimized)

| Location | Python 2D | Simcenter 3D |
| --- | --- | --- |
| **Center** (heated) | **5.66%** | 8.32% |
| **Corner** (unheated) | 7.28% | **6.30%** |

### Error Regime Chronology

1. **$t \approx 9000\text{ s}$** ($2\text{ W} \rightarrow 10\text{ W}$ step): Sharp transient spike up to $\sim 20.8\%$ (phase lag)
2. **$11000\text{--}17000\text{ s}$** ($10\text{ W}$ plateau): Stable error plateau $\sim 4\%$ (excellent agreement)
3. **$t \approx 17000\text{ s}$** ($10\text{ W} \rightarrow 5\text{ W}$ step): Second transient spike
4. **$t > 18000\text{ s}$** (cooldown): Corner drift up to $\sim 15.5\%$ (unmodeled recovery dynamics)

### Simcenter 3D Validation

* Maximum out-of-plane temperature gradient: $\Delta T_z < 0.3\text{ K}$
* **Validates the 2D planar assumption** (aspect ratio $e/L = 0.016$)

### Global Energy Balance Error (RSS)

| Phase | $P_{\text{eff}}$ | $\delta E_{\text{parasitic}}$ | $\delta E_{\text{shroud}}$ | $\delta E_{\text{sensor}}$ | $\delta E_{\text{total}}$ |
| --- | --- | --- | --- | --- | --- |
| $0\text{--}10\text{k s}$ | 1.2 W | 26.34% | 5.21% | 5.34% | **27.4%** |
| $10\text{k}\text{--}17\text{k s}$ | 6.0 W | 5.27% | 1.04% | 5.34% | **7.6%** |
| $17\text{k}\text{--}20\text{k s}$ | 3.0 W | 10.53% | 2.08% | 5.34% | **12.0%** |

---

## Installation & Requirements

### Python Dependencies

```bash
pip install numpy pandas scipy matplotlib casadi openpyxl

```

**Tested with**: Python 3.13, NumPy $\ge 1.26$, SciPy $\ge 1.13$, CasADi $\ge 3.6$

### `requirements.txt`

```text
numpy>=1.26
pandas>=2.2
scipy>=1.13
matplotlib>=3.8
casadi>=3.6
openpyxl>=3.1

```

### Simcenter 3D

The Simcenter 3D model is not included in this repository (commercial license required). Only the exported nodal temperature results (`SimcenterData.xlsx`) are provided for cross-validation.

---

## Usage

1. **Clone the repository**:
```bash
git clone https://github.com/<your-username>/PCB-plate-TVAC-thermal-study.git
cd PCB-plate-TVAC-thermal-study

```


2. **Install dependencies**:
```bash
pip install -r requirements.txt

```


3. **Launch the notebook**:
```bash
jupyter notebook "Plate Analysis.ipynb"

```


4. **Execute cells sequentially**:
* Section 0: Load Simcenter 3D data
* Section 1: Data ingestion, filtering, time normalization
* Section 2: Explicit/implicit solver runs + mesh convergence
* Section 3: Boundary sensitivity analysis (4 cases)
* Section 4: CasADi/IPOPT MDAO parameter identification
* Section 5: Production run with optimized parameters
* Section 6: Cross-correlation alignment + relative error plots
* Section 7: Final comparison plots (Exp vs Python vs Simcenter 3D)



> **Note**: The CasADi MDAO optimization step may take 1–5 minutes depending on hardware (IPOPT iterations + symbolic graph compilation). The production $50 \times 50$ implicit run takes $\sim 30\text{ seconds}$.

---

## Figures & Plots

The notebook generates the following key figures:

| Figure | Description |
| --- | --- |
| Full TVAC campaign | 48-hour raw experimental thermal history |
| Filtered test window | 5.5-hour isolated calibration window |
| Explicit divergence | Numerical blow-up when CFL violated ($50 \times 50, \Delta t = 5\text{ s}$) |
| Implicit convergence | Mesh independence study ($10 \times 10 \rightarrow 200 \times 200$) |
| Boundary sensitivity | 4-case temperature + absolute error comparison |
| MDAO optimization | Relative error before/after parameter identification |
| Aligned center | Exp vs Python vs Simcenter (center node, cross-correlated) |
| Aligned corner | Exp vs Python vs Simcenter (corner node, cross-correlated) |
| Relative error center | Python vs Exp + Simcenter vs Exp over time |
| Relative error corner | Python vs Exp + Simcenter vs Exp over time |
| Degradation bars | RMSE penalty from shroud/power approximations |

---

## References

Key references for the numerical methods, TVAC testing, and thermal modeling approaches used:

* Bergman et al., *Fundamentals of Heat and Mass Transfer* (8th ed.), Wiley, 2017
* Patankar, *Numerical Heat Transfer and Fluid Flow*, Hemisphere, 1980
* Andersson et al., *CasADi — A software framework for nonlinear optimization and optimal control*, Mathematical Programming Computation, 2019
* Gilmore, *Spacecraft Thermal Control Handbook*, AIAA, 2002
* ECSS-E-HB-31-03A, *Thermal Analysis Handbook*, ESA-ESTEC, 2016

Full bibliography available in the accompanying IEEE conference paper.

---

## License

This project is part of an academic research study conducted at **ISAE-ENSMA / ÉTS Montréal**.

© 2026 Luca Ségala. All rights reserved.

For academic collaboration or data access inquiries, please contact the author.
