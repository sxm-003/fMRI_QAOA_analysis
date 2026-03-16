# fMRI to QAOA Analysis

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

## End-to-End Notebook Workflow

## 1) Load and inspect fMRI volume

The notebook starts by loading a 4D NIfTI file using `nibabel`:

- `img = nib.load(...)`
- `data = img.get_fdata()`

Then it checks shape and visualizes slices:

- static axial slice with `matplotlib`
- interactive volume viewer with `nilearn.plotting.view_img`

Purpose:

- sanity-check spatial and temporal dimensions,
- verify data quality visually before modeling.

## 2) Read trial/event table

A TSV is loaded with `pandas.read_csv(..., sep='\t')`.

This gives task timing metadata (`events.tsv`) that can later support task-aware analyses, even though the main current pipeline focuses on ROI signal/connectivity extraction.

## 3) Atlas-based ROI time-series extraction

You fetch Schaefer 2018 atlas (`n_rois=200`) and apply `NiftiLabelsMasker` with denoising options:

- `standardize=True`
- `detrend=True`
- `smoothing_fwhm=6`
- `low_pass=0.1`
- `high_pass=0.01`
- `t_r=2`

Then:

- `time_series = masker.fit_transform(img)`

This yields a matrix of shape `(T, N)` where:

- `T` = number of time points,
- `N` = number of atlas parcels retained.

The notebook also visualizes example ROI signals over time.

## 4) Functional connectivity construction

Connectivity is computed via Pearson correlations between ROI time series:

- `corr = np.corrcoef(time_series.T)`

You then:

- set diagonal to zero,
- normalize by max absolute value,
- apply hard threshold (`0.25`) to sparsify weak edges.

This produces a weighted, sparse-like ROI interaction matrix used as Ising couplings.

## 5) ROI activity vector

Per-ROI mean activity is computed and normalized:

- `mean_activity = np.mean(roi_ts, axis=0)`
- divide by max absolute value.

This vector becomes the linear bias component in the Hamiltonian.

## 6) Build Ising Hamiltonian (`SparsePauliOp`)

You create:

- `Z_i` terms with coefficients from `mean_activity[i]`,
- `Z_i Z_j` terms from nonzero `corr[i, j]`.

Then construct:

- `H = SparsePauliOp(terms, coeffs)`

Interpretation:

- Each ROI corresponds to one qubit in this formulation.
- The energy landscape encodes both local region tendencies and pairwise interactions.

## 7) ROI ↔ qubit mapping table

You explicitly map atlas parcel IDs to qubit indices, decode atlas labels, and build a readable table:

- `roi_to_qubit`
- `qubit_to_roi`
- `roi_qubit_table`

This is important for interpreting sampled bitstrings in neuroanatomical terms.

## 8) QAOA setup on IBM Quantum Runtime

The notebook loads credentials from environment variables (and optionally `.env` files):

- `IBM_QUANTUM_TOKEN` (required)
- `IBM_QUANTUM_INSTANCE` (optional/instance dependent)

Then:

- connects to `QiskitRuntimeService` on `ibm_cloud`,
- selects backend (`ibm_fez`),
- constructs `QAOAAnsatz(H, reps=2)`,
- transpiles ansatz and aligned Hamiltonian (`H_isa`),
- builds measured circuit for sampling.

## 9) Hybrid optimization loop (COBYLA + Estimator)

A classical optimizer (`scipy.optimize.minimize`, COBYLA) minimizes the objective by repeatedly evaluating:

- expectation value of `H_isa` for current QAOA parameters

using:

- `EstimatorV2(mode=backend)`

The notebook tracks:

- objective history (`objective_history`),
- parameter history (`param_history`),
- iteration history (`iteration_history`),
- summary DataFrames (`opt_trace`, `iter_trace`).

## 10) Sampling final distribution (Sampler)

With optimized (or selected) parameters, the measured QAOA circuit is sampled via:

- `SamplerV2(mode=backend)`

You obtain:

- raw `counts` dictionary,
- top 50 bitstrings with probabilities (`top_50_bitstrings`).

These are candidate low-energy solutions in the encoded brain-network Ising landscape.

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
- QAOA depth (`reps=2`) and COBYLA iterations (`maxiter=25`) are sufficient for a first pass.

These are tunable choices and can be expanded for sensitivity analyses.

---

## Suggested Next Extensions

- multi-subject aggregation and subject-specific comparison,
- threshold sweep / regularization strategy for couplings,
- objective rescaling and coefficient calibration,
- larger/smaller ROI sets to study qubit-resource tradeoffs,
- interpretation layer mapping top bitstrings back to ROI labels and network motifs.

---

## Summary

This notebook operationalizes a neuro-to-quantum workflow:

- from voxel-level BOLD data
- to ROI network representation
- to Ising Hamiltonian encoding
- to hybrid quantum-classical optimization with QAOA
- to interpretable sampled bitstring solutions.

It serves as a strong prototype for exploring quantum formulations of functional brain network structure.