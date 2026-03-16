# fMRI to QAOA Analysis [In Progress , Open for Collaborations]

This project builds a full pipeline from **fMRI data** to a **QAOA optimization problem**:

1. Load and inspect 4D BOLD fMRI data.
2. Extract ROI-level time series using a Schaefer atlas.
3. Build a functional connectivity matrix.
4. Convert brain activity/connectivity into an Ising Hamiltonian (`SparsePauliOp`).
5. Run QAOA on an IBM Quantum backend.
6. Sample bitstrings and inspect most probable solutions.

The full workflow is implemented in:

- `data_extract.ipynb`

---

## Project Goal

You are translating brain dynamics into a quantum optimization formulation.

- **Neuro side**: ROI activity and pairwise functional connectivity from BOLD signals.
- **Quantum side**: Ising cost Hamiltonian with:
  - single-qubit `Z_i` bias terms from mean ROI activity,
  - two-qubit `Z_i Z_j` coupling terms from ROI correlations.
- **Optimization side**: QAOA parameters are tuned via classical COBYLA using quantum expectation evaluations.

---

## Dataset and Inputs

Primary data source:

- `fMRI_data/ds000003_R2.0.2/`

Notebook currently demonstrates processing on:

- Subject BOLD image: `sub-01/func/sub-01_task-rhymejudgment_bold.nii.gz`
- Subject events table: `sub-01/func/sub-01_task-rhymejudgment_events.tsv`

Additional metadata and quality files exist under dataset directories (BIDS-style structure).

---

## Key Variables and What They Mean

- `img`, `data`: raw 4D fMRI image and voxel data.
- `time_series`: denoised ROI-by-time matrix.
- `corr`: ROI-ROI functional connectivity matrix.
- `mean_activity`: normalized ROI mean activation vector.
- `H`: Ising Hamiltonian as `SparsePauliOp`.
- `ansatz`, `ansatz_isa`, `H_isa`: logical and backend-transpiled QAOA components.
- `result`: COBYLA optimization result.
- `counts`, `top_50_bitstrings`: sampled outcome distribution.

---

## Environment Requirements

Python packages used in the notebook include:

- `numpy`
- `pandas`
- `matplotlib`
- `nibabel`
- `nilearn`
- `scipy`
- `qiskit`
- `qiskit-ibm-runtime`

For IBM Runtime execution:

- valid IBM Quantum credentials,
- network access,
- access entitlement for selected backend.

---

## How to Run

1. Open `data_extract.ipynb`.
2. Execute cells top-to-bottom in order.
3. Ensure ROI extraction and Hamiltonian construction cells complete before QAOA cells.
4. Set `IBM_QUANTUM_TOKEN` (and `IBM_QUANTUM_INSTANCE` if required) before runtime cells.
5. Run optimization cell and then sampling cells.
6. Inspect `opt_trace`, `iter_trace`, and `top_50_bitstrings`.

---

## Current Modeling Assumptions

- ROI-mean activity can be represented as linear Ising biases.
- Pearson correlation captures pairwise coupling strength.
- Hard-thresholding at `0.25` is acceptable sparsification.
- QAOA depth (`reps=2`) and COBYLA iterations (`maxiter=25`) are insufficient for a first pass.

These are tunable choices and can be expanded for sensitivity analyses.

---


## Summary

This notebook operationalizes a neuro-to-quantum workflow:

- from voxel-level BOLD data
- to ROI network representation
- to Ising Hamiltonian encoding
- to hybrid quantum-classical optimization with QAOA
- to interpretable sampled bitstring solutions.

It serves as a prototype for exploring quantum formulations of functional brain network structure.