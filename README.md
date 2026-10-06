# Golden Signals for AI Workloads on Shared GPU Clusters

**PyTorch Conference North America 2026 poster.** *[What Should SRE Watch? Calibration-Gated Decisions and the Missing Golden Signals for GPU Clusters](https://events.linuxfoundation.org/pytorch-conference-north-america/program/schedule-posters/?id=1293358)* — Community Expo, Tuesday 20 October, 6:00–7:30pm.

This repository is the poster and taxonomy home. Measured queue-reliability evidence lives in companion repos. The live operational surface is [VGAC](https://demo.vgac.cloud/).

| Artifact | Audience | Status |
| --- | --- | --- |
| **Poster** (`tex/poster/poster.pdf`) | PyTorch practitioners / SREs | First full draft --- **24×34 in portrait** (fits within organizer 36×24 max) |
| **Research paper** (`tex/research/`) | ML systems / SRE researchers | Outline + skeleton |
| **Practitioner article** (`tex/practitioner/`) | SREs, ML infra operators | Outline + skeleton |

## The problem

The four [golden signals of monitoring](https://sre.google/sre-book/monitoring-distributed-systems/) — latency, traffic, errors, saturation — were written for stateless request/response services. On shared GPU clusters they break down: latency has token / batch / epoch / queue meanings; QPS is undefined for training; HTTP errors miss silent quality failure; `nvidia-smi util%` is a time-busy metric, not useful compute.

## Five candidate signals

These are a **minimum viable set** for a paging rotation, not a closed empirical claim. Only QWR is measured.

| Signal | Generalizes | Evidence in this program |
| --- | --- | --- |
| **QWR** — Queue-of-Work Reliability | Latency (queue) | **Measured.** Held-out Slurm study + live VGAC `/model/health` |
| **GCE** — GPU-Compute Effectiveness | Saturation (compute) | Defined. DCGM fields exist; the Slurm T4 run was idle (`util=0`) |
| **HPH** — HBM-Pressure Headroom | Saturation (memory) | Defined. Same idle-trace limit |
| **SAI** — Straggler Amplification Index | Latency (collectives) | Candidate. No per-worker step times yet |
| **OQC** — Output-Quality Calibration | Errors (quality) | Candidate. Do not confuse with queue-delay ECE (that is QWR) |

Definitions and SLO templates: [`docs/SIGNALS_TAXONOMY.md`](docs/SIGNALS_TAXONOMY.md), [`docs/SLO_TEMPLATES.md`](docs/SLO_TEMPLATES.md). Why the classical four fail: [`docs/CRITIQUE.md`](docs/CRITIQUE.md).

## What you can cite today

Poster numbers are the **held-out** Slurm split and the live model card. A local-only `results/model_evaluation_slurm.json` reports GB AUROC 0.985 / ECE 0.006 on 388/167; that file is **not** on GitHub and looks in-sample. Do not print it.

| Claim | Number | Source |
| --- | --- | --- |
| Slurm GB, held-out | AUROC **0.863** (CI 0.71–0.96), ECE **0.056**, n_test=111 (444/111 of 555), tier **warn** (T2: ECE < 0.07 but > 0.05) | [Reliability-First-Queue-Risk](https://github.com/espirado/Reliability-First-Queue-Risk) `artifacts/slurm_real_benchmark.json` |
| Slurm label | long wait = wait > P90 = 133s, ~10% base rate; ~8 positives in the test set; 0 true alerts at p≥0.7 | same JSON + Paper 2 wait table |
| Live VGAC production | AUROC 0.766, ECE **0.068**, n=5,184, `v4.0-richfeatures-lr` | [demo.vgac.cloud](https://demo.vgac.cloud/) `/model/health` |
| Paper 3 VGAC abstract | 650 EKS jobs, AUROC 0.756, ECE 0.077 | `Prediction-to-Policy-Integration` |
| EKS label | long wait = wait > 120s, near the median (~48% violation rate in the 650-job notebook) | Paper 3 / `paper2_notebook_results.json` |

**The gate does not pass production.** Advisory is ECE ≤ 0.05; gate is ECE ≤ 0.03. Live 0.068 and Paper 3 0.077 are both **T2 Warn** on the Annotate/Warn/Suggest/Gate ladder. Say that on a poster titled *Calibration-Gated Decisions*. Do not use `sli_dashboard.png` panel (d), which paints EKS as T4 Gate.

Do **not** print “1.2M pod events validate five signals,” “22% goodput lost to stragglers,” “util% vs goodput r≈0.1,” or drift PSI≈12.4. Those are literature/simulation, undefined on idle DCGM, or empty-bin artifacts (`data/samples/drift_metrics.json`: ECE 0.000 and AUROC 1.000 in seven of eight windows).

## Figures

| File | Print? |
| --- | --- |
| `calibration_curve.png` (ECE 0.077) | Yes |
| `wait_distribution.png` | Yes |
| `wait_vs_queue_depth.png` | Yes |
| `sli_dashboard.png` | No — (d) wrong EKS tier, (e) degenerate ECE, (f) empty |
| `transfer_and_drift.png` | No — PSI ~12.41 empty-bin artifact |
| `reliability_diagrams.png` | Only with a caption: EKS n=4,000 on that plot vs 650 / 5,184 elsewhere |

Wherever Slurm and EKS appear together, state the two different “long wait” labels.

## Live demo

- UI: https://demo.vgac.cloud/
- Deploy tree: private repo [espirado/vgac](https://github.com/espirado/vgac) — `scripts/deploy-demo.sh`, `infra/terraform/envs/demo`

VGAC tabs map to the poster: Predictions = QWR, Calibration = the gate (currently **warn**), GPUs = GCE/HPH candidates, HPC = SAI candidate.

## Companion repositories

- [Reliability-First-Queue-Risk](https://github.com/espirado/Reliability-First-Queue-Risk) — Paper 2, QWR SLIs. Public tree has `artifacts/` and `data/samples/`, not `results/` or `data/training_dataset.csv`.
- [Prediction-to-Policy-Integration](https://github.com/espirado/Prediction-to-Policy-Integration) — Paper 3 / VGAC policy
- `research-fall2025` — shared `src/sli` and `src/tier` (local data volume may be unmounted)

## Authors

- Andrew Espira, Saint Peter's University — [ORCID 0009-0002-9196-8094](https://orcid.org/0009-0002-9196-8094)
- Sharath Kumar, Saint Peter's University (companion papers; confirm Sessionize listing before printing the poster)

**Community context.** The framing — that AI GPU workloads lack an accepted golden-signal set — draws on public Google SRE community discussions on monitoring AI workloads in large production deployments.
