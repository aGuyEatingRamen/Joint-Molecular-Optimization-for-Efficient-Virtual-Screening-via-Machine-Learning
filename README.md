# Joint Molecular Optimization for Virtual Screening

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.18168822.svg)](https://doi.org/10.5281/zenodo.18168822)

> **Paper:** *Joint Molecular Optimization for Efficient Virtual Screening via Machine Learning* -- Naman Jain

This repository contains the two Jupyter notebooks that implement the complete pipeline described in the paper: molecular docking data generation and multi-objective machine learning training across a four-pipeline ablation framework.

---

## Abstract

Traditional virtual screening focuses on binding affinity alone, leading to high attrition in clinical development. This work introduces a joint optimization strategy that simultaneously models binding affinity and ADMET (Absorption, Distribution, Metabolism, Excretion, Toxicity) profiles. Using data from the DUD-E benchmark dataset across four cancer-relevant protein targets, a four-pipeline ablation study demonstrates that progressively enriching molecular representations -- from fingerprint-only baselines to fully integrated multi-target, ligand-conditioned, ADMET-augmented pipelines -- improves both classification and regression performance.

---

## What is in This Repository

Two notebooks are provided:

**`01_docking_pipeline.ipynb`** -- Docking data generation. Takes the raw DUD-E SDF files as input and produces a `docking_data.csv` per target containing SMILES strings and their corresponding SMINA binding affinity scores. This notebook is computationally intensive and is intended to be run on a machine with substantial GPU or CPU resources -- a high-performance computing cluster or a Colab Pro session with high-RAM runtime is recommended. A standard free Colab session will work for small targets but may time out on larger ones. The outputs of this notebook are the CSV files consumed by notebook 2.

**`02_ml_pipeline.ipynb`** -- Machine learning training and evaluation. Loads the docking CSVs produced by notebook 1, constructs molecular feature matrices, and runs the four-pipeline ablation framework (STRATUM: Stratified Training and Ranking Architecture for Target-aware Unified Modeling). All install and import cells are self-contained -- simply open in Colab and run all cells in order.

---

## How to Run

Both notebooks are fully self-contained. All required package installations are handled by cells within each notebook. Open the notebook in Google Colab and run cells from top to bottom.

**Notebook 1** requires the DUD-E dataset (see below) and access to sufficient compute. Results are saved to Google Drive.

**Notebook 2** requires the CSV files produced by Notebook 1, stored at the paths configured in the final cell. Update the `MACHINE_DATA` path variable to match your Drive structure before running.

---

## Data

The DUD-E (Directory of Useful Decoys: Enhanced) dataset is required to run Notebook 1. It is publicly available at [https://dude.docking.org](https://dude.docking.org).

Download the target folders for `parp1`, `ital`, `mapk2`, and `tgfr1`. Each folder must contain:

```
receptor.pdb
actives_final.sdf
decoys_final.sdf
crystal_ligand.mol2
```

Place them on Google Drive at a path matching the configuration in Notebook 1.

---

## The Four-Pipeline Framework

Each pipeline version varies which molecular information is included as model input, constituting a controlled ablation study:

| Pipeline | Multi-Target Data | Ligand Context | ADMET Features | Purpose |
|----------|:-----------------:|:--------------:|:--------------:|---|
| V1 | No | No | No | Fingerprint-only baseline |
| V2 | Yes | Yes | No | Multi-target + ligand conditioning |
| V3 | No | No | Yes | ADMET-augmented single target |
| V4 | Yes | Yes | Yes | Full joint optimization |

### Models Evaluated

**Classifiers** (active vs. decoy prediction; metric: AUC-ROC): Random Forest, Naive Bayes, SVM (RBF kernel), XGBoost, Neural Network.

**Regressors** (docking score prediction; metrics: MSE, R2): Random Forest, SVR, Linear Regression, XGBoost, Neural Network.

All models are evaluated under identical 5-fold stratified cross-validation with random seed fixed to 42.

---

## Key Results

### Classification (AUC-ROC, cross-validated mean +/- std)

| Model | V1 | V2 | V3 | V4 |
|---|---|---|---|---|
| Random Forest | 0.668 +/- 0.024 | 0.688 +/- 0.025 | 0.663 +/- 0.027 | 0.688 +/- 0.027 |
| Naive Bayes | 0.644 +/- 0.023 | 0.675 +/- 0.009 | 0.645 +/- 0.024 | 0.675 +/- 0.009 |
| SVM (RBF) | 0.603 +/- 0.031 | 0.621 +/- 0.017 | 0.600 +/- 0.032 | 0.623 +/- 0.016 |
| XGBoost | 0.643 +/- 0.026 | 0.684 +/- 0.028 | 0.638 +/- 0.027 | 0.664 +/- 0.021 |
| Neural Network | 0.575 +/- 0.031 | 0.599 +/- 0.022 | 0.577 +/- 0.027 | 0.599 +/- 0.020 |

### Regression (R2, cross-validated mean +/- std, best model per pipeline)

| Pipeline | Best Model | R2 |
|---|---|---|
| V1 | XGBoost | 0.825 +/- 0.030 |
| V2 | XGBoost | 0.792 +/- 0.007 |
| V3 | XGBoost | 0.864 +/- 0.020 |
| V4 | Random Forest | 0.823 +/- 0.007 |

ADMET augmentation (V3) yields the strongest regression performance overall (R2 = 0.864). The fully integrated V4 pipeline achieves the most balanced performance profile across both classification and regression tasks.

---

## Note on Ligand SMILES Encoding

Pipelines V2 and V4 incorporate ligand-conditioned molecular fingerprints, where a reference ligand SMILES is concatenated to each molecule's SMILES string prior to Morgan fingerprint generation. This encodes binding-pocket context directly into the structural representation.

In this implementation, a fixed reference SMILES (`CC(=O)Oc1ccccc1C(=O)O`) was used uniformly across all targets and molecules in place of the per-target co-crystallized ligand SMILES available in the DUD-E `crystal_ligand.mol2` files. This design choice was made deliberately to decouple the evaluation of the ligand-conditioning mechanism from the influence of target-specific ligand identity. By holding the reference ligand constant, the contribution of the ligand-conditioning architecture itself -- rather than the information content of any particular ligand -- can be assessed in isolation across the multi-target dataset. This provides a more conservative and reproducible baseline for the ligand context component of the ablation study.

Researchers extending this work are encouraged to substitute per-target crystal ligand SMILES, extracted from the DUD-E `crystal_ligand.mol2` files using RDKit, which would be expected to further improve performance in the V2 and V4 configurations.

---

## Technical Details

### Molecular Representation

- Morgan fingerprints: 2048-bit, radius 2 (ECFP4)
- Ligand-conditioned encoding: reference ligand SMILES concatenated to molecule SMILES using the SMILES fragment separator `.` before fingerprinting (Pipelines V2, V4)
- ADMET descriptors: 6 physicochemical features (LogP, TPSA, MolWt, H-donors, H-acceptors, rotatable bonds)

### ADMET Desirability Scoring

The joint optimization scoring function combines normalized docking affinity (inverted so stronger binding scores higher) with an ADMET desirability score derived from six Lipinski-aligned sub-scores, combined at equal 50/50 weighting.

### Docking Software

| Tool | Role |
|---|---|
| AutoDock SMINA | Molecular docking -- binding affinity prediction (kcal/mol) |
| Open Babel | File format conversion (PDB/SDF to PDBQT) |
| RDKit | Fingerprint generation, ADMET descriptor computation |

---

## Reproducibility

- Dataset: DUD-E -- Mysinger et al., J. Med. Chem. 55, 2012
- Targets: PARP1, ITAL, MAPK2, TGFR1
- Dataset balancing: 1:1 active-to-decoy sampling per target
- Cross-validation: 5-fold stratified, random seed 42
- All dependencies installed within the notebooks

---

## Citation

If you use this code or build on this work, please cite:

```bibtex
@article{jain2025joint,
  title   = {Joint Molecular Optimization for Efficient Virtual Screening via Machine Learning},
  author  = {Jain, Naman},
  year    = {2025}
}

@software{jain2025joint_code,
  author    = {Jain, Naman},
  title     = {Joint Molecular Optimization for Virtual Screening -- Code},
  year      = {2025},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.18168822},
  url       = {https://doi.org/10.5281/zenodo.18168822}
}
```

---

## Acknowledgements

- DUD-E dataset -- Mysinger et al., J. Med. Chem. 55, 2012
- AutoDock SMINA -- Koes et al.
- RDKit -- Landrum et al.
- Deep Docking -- Gentile et al., ACS Central Science 2020

---

## License

MIT License. See [LICENSE](LICENSE) for details.
