# SpatialTME-AI — Findings

Husain Naqvi · Project 15 · November 2026

Every number below comes from a file in `results/`, and the file is named next to the result.

## 1. Question and design

Spot-based spatial transcriptomics such as Visium (55 µm spots) needs **deconvolution** to recover cell-type composition. Methods are usually compared on simulated spots. I built a benchmark from real tissue instead.

**Data.** Janesick et al. 2023 (GEO GSE243280) profiled one breast tumour with:
- Xenium in two adjacent sections, with cell-type labels for each cell;
- Visium CytAssist;
- 3′ and 5′ scRNA-seq.

**Ground truth.** Labelled Xenium cells inside each Visium-geometry spot are summed (hexagonal lattice, 100 µm spacing; spots of 55 µm, plus a 100 µm version). Spots need ≥ 3 cells and ≥ 60% labelled. The true composition over 8 types is the fraction of labelled cells: tumour, myoepithelial, fibroblast, endothelial, perivascular, myeloid, T and B.

**Protocol.**
- Replicate 1 (`xr1_*`) was the development set for every choice.
- Replicate 2 (`xr2_*`) was held out and scored once with frozen settings.
- Robustness was tested at 100 µm spots and with the 5′ reference.

**Metrics.**
- Per-spot RMSE and JSD.
- Mean per-type Pearson r over spots. This weights rare immune and vascular types equally, so it matters most for mapping the microenvironment.
- A spatial block bootstrap (600 µm tiles, 500 resamples) compared every method with GraphDecon on the same spots (`benchmark_bootstrap_vs_graphdecon.csv`).

**Methods.**
- **SpotGNN (pre-specified).**
  - An encoder with two residual graph layers over the spot lattice.
  - Trained on 24 simulated tissues laid out on the target's own lattice, with spatially smooth composition fields and spots synthesised from reference cells.
  - A gradient-reversal discriminator aligns simulated and real spots.
  - Ablations: no graph layers, no domain adaptation, and an MLP trained on unstructured simulations.
- **SpotGNN v2 (post hoc).** Raw-count simulation, shared normalisation and platform factors, tried after v1 lost on replicate 1.
- **GraphDecon.** Counts follow y ~ Poisson(library × mixture of reference profiles × per-gene platform factor), with an optional spatial-smoothness term. It is fitted per tissue (Adam, NNLS initialisation) and needs no labels or simulation. λ was tuned on replicate 1 (`graphdecon_tuning_rep1.csv`).
- **Baselines:** NNLS, NNLS + platform factors, cell2location, Stereoscope and Tangram, all with tutorial settings.

## 2. Result 1: held-out benchmark (`benchmark_metrics.csv`)

Replicate 2, 55 µm, 3′ reference (2,715 spots):

| Method | RMSE | JSD | Mean r | Δr vs GraphDecon (95% CI) |
|---|---|---|---|---|
| GraphDecon | 0.094 | 0.091 | **0.852** | — |
| GraphDecon w/o platform | 0.092 | 0.089 | 0.854 | −0.002 (−0.004, −0.001) |
| cell2location | 0.111 | 0.104 | 0.830 | +0.022 (0.014, 0.030) |
| NNLS + platform | 0.090 | 0.084 | 0.819 | +0.033 (0.025, 0.042) |
| NNLS | 0.094 | 0.088 | 0.798 | +0.053 (0.043, 0.065) |
| Stereoscope | 0.134 | 0.140 | 0.777 | +0.074 (0.060, 0.088) |
| MLP (unstructured sim.) | 0.158 | 0.170 | 0.727 | +0.124 |
| Tangram | 0.164 | 0.181 | 0.725 | +0.127 |
| SpotGNN | 0.203 | 0.249 | 0.678 | +0.174 |
| SpotGNN v2 | 0.259 | 0.386 | 0.338 | +0.517 |

**Robustness.**

| Setting | GraphDecon r | cell2location r | Δr (95% CI) |
|---|---|---|---|
| 100 µm spots | 0.845 | 0.809 | +0.037 (0.028, 0.046) |
| 5′ reference | 0.845 | 0.798 | +0.047 (0.039, 0.055) |

NNLS has the lowest RMSE at 100 µm (0.083), with r 0.779.

**Per type (r on replicate 2).**
- GraphDecon: tumour 0.96, myoepithelial 0.92, fibroblast 0.91, endothelial 0.89, perivascular 0.67, myeloid 0.76, T 0.86, B 0.85.
- cell2location is better for tumour (0.98) and perivascular cells (0.78), and worse for T (0.72) and myeloid cells (0.69).
- NNLS is weakest on B (0.65) and myoepithelial cells (0.75).

## 3. Result 2: what did and did not help

- **SpotGNN failed, and this is reported as a pre-specified negative result.** It was trained to fit simulations, and the domain adversary aligned the embeddings, yet the predictions on real spots were biased.
  - Removing the graph layers or the domain adaptation changed little.
  - The structured simulations did worse than unstructured ones (MLP).
  - The post hoc v2 was worse still. I did not keep revising it on held-out data.
- **Spatial smoothing was not selected.** On replicate 1, RMSE was lowest at λ = 0 (0.090), and λ = 1 gave 0.101. A small λ (0.03–0.1) raised r slightly (0.830 vs 0.822), but the selection rule was RMSE and fixed in advance, so λ = 0 was frozen.
- **Platform factors are real but irrelevant here.** The learned log factors have median |b| 0.66, and 31% of genes are above 1 (`graphdecon_meta.json`). Removing them changes GraphDecon by less than 0.003. Adding them to NNLS helps (r 0.798 to 0.819) but does not reach GraphDecon. **The gain comes from the Poisson count likelihood.**

## 4. Result 3: real Visium (`visium_vs_registered_xenium.csv`, `visium_registration.json`)

**Registration.** Visium was registered to the adjacent Xenium replicate 1 by expression alone (tumour-minus-stroma score, rotation/reflection/scale grid, FFT cross-correlation r = 0.56). Each Visium spot received the composition of the Xenium cells under it (3,242 spots).

| Method | Mean r | RMSE |
|---|---|---|
| cell2location | 0.175 | 0.191 |
| Stereoscope | 0.160 | 0.208 |
| GraphDecon | 0.153 | 0.198 |
| SpotGNN | 0.132 | 0.250 |
| Tangram | 0.131 | 0.239 |
| NNLS | 0.129 | 0.243 |

- Agreement is low for every method. Only tumour (r 0.32–0.49) and fibroblast (0.23–0.44) are recovered.
- The two Bayesian methods rank above GraphDecon.
- The reference here is weak: adjacent section, partial registration, different chemistry. Even so, the ranking on real Visium differs from the ground-truth ranking.

## 5. Niches (`visium_niches.csv`)

k-means (k = 6) on neighbourhood-averaged GraphDecon compositions:

| Niche | Spots | Defining composition |
|---|---|---|
| Tumour core | 635 | tumour 50% |
| Mixed tumour–stroma | 803 | tumour 28%, fibroblast 24% |
| Mixed, B-cell-rich | 786 | B 26% |
| Dense stroma | 809 | fibroblast 65% |
| Vascular stroma | 861 | endothelial 17% |
| Stroma, B-cell-rich | 1,098 | fibroblast 47%, B 18% |

## 6. What I would claim, and what I would not

**Claim:**
- On real tissue with known composition, a Poisson likelihood model gives the best per-type accuracy of the methods tested and ties the best RMSE among them.
- A simulation-trained, domain-adapted GNN does not transfer.
- Smoothing and platform correction add nothing here.

**Not claim:**
- Generality beyond one tumour and a 313-gene panel.
- Superiority on real Visium, where cell2location ranked first.
- That GraphDecon is a new state of the art. It is a simple model, and its spatial term was switched off by the tuning rule.

## 7. Reproducibility

- **Compute:** all code runs on Google Colab.
- **Data:** downloaded from GEO GSE243280 by `src/st.py`.
- **Seeds:** fixed.
- **Baselines:** run in an isolated environment with per-job lock files.
- **Figures:** Figs 4–5 were re-drawn from `results/` by `scripts/figures_local.py`.
