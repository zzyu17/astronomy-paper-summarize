---
name: astronomical_pipeline
description: "7-phase astronomical research pipeline — context knowledge for methodology_analyst_agent and deep-reading agents"
version: "1.0.0"
---

# The 7-Phase Astronomical Research Pipeline

Modern astronomical research typically flows through seven interconnected phases. This pipeline is **context knowledge** for agents — not a rigid mapping. Each paper may cover some, many, or all phases. Rich feedback loops exist between all phases.

## Pipeline Phases

| Phase | Description | Key Activities | Typical Outputs |
|-------|-------------|---------------|-----------------|
| **1. Theory & Physical Modeling** | Mathematical/analytical framework | Deriving physical equations, building conceptual models (ΛCDM, core accretion, MRI), making predictions | Equations, scaling relations, analytical predictions |
| **2. Numerical Simulation** | Computational implementation | N-body, hydro, MHD, radiative transfer; large simulation suites (IllustrisTNG, EAGLE, CAMELS) | Simulated catalogs, mock observations, parameter spaces |
| **3. Observational Design & Proposal** | Planning observations | Target selection, feasibility calculations, instrument selection, exposure time estimation, proposal writing | Observing proposals, target lists, feasibility reports |
| **4. Data Acquisition** | Collecting data | Imaging, spectroscopy, interferometry, time-domain surveys (TESS, JWST, ALMA, VLA, SDSS, LSST) | Raw data files (FITS, HDF5) |
| **5. Data Reduction & Calibration** | Processing raw data | Bias/flat/dark correction, wavelength/flux calibration, astrometric calibration, source extraction | Reduced data products, catalogs |
| **6. Statistical Inference & Analysis** | Extracting physical parameters | MCMC, nested sampling, SED fitting, machine learning classification, Bayesian inference, population statistics | Parameter posteriors, HR diagrams, mass functions |
| **7. Interpretation & Model Comparison** | Connecting to physics | Bayes factors, posterior predictive checks, theory-data comparison, physical interpretation | Scientific conclusions, theory constraints |

## Key Insight

All phases are **bidirectionally connected**. The pipeline is a web, not a line:
- Simulations (Phase 2) inform observational design (Phase 3) and interpret results (Phase 7)
- Observations (Phase 4) constrain theory (Phase 1) and drive new simulations (Phase 2)
- Statistical methods (Phase 6) evolve based on data characteristics (Phase 5) and theoretical requirements (Phase 1)

## Subfield Variations

| Subfield | Typical Phase Emphasis |
|----------|----------------------|
| **Exoplanets** | 3-4-5-6 (observation-heavy, transit/radial velocity pipelines) |
| **Cosmology** | 1-2-6-7 (theory + simulation + statistical inference) |
| **Galaxy Evolution** | 2-4-5-6 (simulation + multi-wavelength surveys) |
| **Stellar Astrophysics** | 4-5-6-7 (spectroscopy + stellar models) |
| **High-Energy Astrophysics** | 4-5-6 (time-domain + multi-messenger) |

## Agent Usage

When the `methodology_analyst_agent` analyzes a paper:
1. Recognize which phases the paper covers (most papers cover 2-4 phases)
2. Use phase awareness when explaining methods — help the user understand where this work fits
3. Do NOT force every paper into all seven phases
4. Do NOT use this as a rigid checklist — it's background context
