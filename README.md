# Golden Signals for AI Workloads on Shared GPU Clusters

**PyTorch Conference North America 2026 poster.** *[What Should SRE Watch? Calibration-Gated Decisions and the Missing Golden Signals for GPU Clusters](https://events.linuxfoundation.org/pytorch-conference-north-america/program/schedule-posters/?id=1293358)* — Community Expo, Tuesday 20 October, 6:00–7:30pm.

This repository is the poster and taxonomy home. Measured queue-reliability evidence lives in companion repos. The live operational surface is [VGAC](https://demo.vgac.cloud/).

| Artifact | Audience | Status |
| --- | --- | --- |
| **Poster** (this repo + `figures/`) | PyTorch practitioners / SREs | In preparation — PDF due 7 October |
| **Research paper** (`tex/research/`) | ML systems / SRE researchers | Outline + skeleton |
| **Practitioner article** (`tex/practitioner/`) | SREs, ML infra operators | Outline + skeleton |

## The problem

The four [golden signals of monitoring](https://sre.google/sre-book/monitoring-distributed-systems/) — latency, traffic, errors, saturation — were written for stateless request/response services. On shared GPU clusters they break down: latency has token / batch / epoch / queue meanings; QPS is undefined for training; HTTP errors miss silent quality failure; `nvidia-smi util%` is a time-busy metric, not useful compute.

## Five candidate signals

These are a **minimum viable set** for a paging rotation, not a closed empirical claim.

| Signal | Generalizes | Evidence in this program |
| --- | --- | --- |
| **QWR** — Queue-of-Work Reliability | Latency (queue) | **Measured.** Slurm 555-job study + live VGAC `/model/health` and `/api/predict/wait` |
| **GCE** — GPU-Compute Effectiveness | Saturation (compute) | Defined. DCGM fields exist; the Slurm T4 run was idle (`util=0`) |
| **HPH** — HBM-Pressure Headroom | Saturation (memory) | Defined. Same idle-trace limit |
| **SAI** — Straggler Amplification Index | Latency (collectives) | Candidate. No per-worker step times yet |
| **OQC** — Output-Quality Calibration | Errors (quality) | Candidate. Do not confuse with queue-delay ECE (that is QWR) |

Definitions and SLO templates: [`docs/SIGNALS_TAXONOMY.md`](docs/SIGNALS_TAXONOMY.md), [`docs/SLO_TEMPLATES.md`](docs/SLO_TEMPLATES.md). Why the classical four fail: [`docs/CRITIQUE.md`](docs/CRITIQUE.md).

## What you can cite today

| Claim | Number | Source |
| --- | --- | --- |
| Slurm long-wait classifier (GB) | AUROC 0.985, ECE 0.006, n=555, P90 wait 133s | [Reliability-First-Queue-Risk](https://github.com/espirado/Reliability-First-Queue-Risk) `results/model_evaluation_slurm.json` |
| Live VGAC production model | AUROC 0.766, ECE 0.068, n=5,184 | [demo.vgac.cloud](https://demo.vgac.cloud/) `/model/health` (`v4.0-richfeatures-lr`) |
| Paper 3 VGAC abstract | 650 EKS jobs, AUROC 0.756, ECE 0.077 | `Prediction-to-Policy-Integration` |

Do **not** print “1.2M pod events validate five signals,” “22% goodput lost to stragglers,” or “util% vs goodput r≈0.1” from this repo. Those either live in another tree, are literature/simulation, or are undefined on the idle Slurm DCGM columns.

Poster-ready plots from Paper 2 are copied into `figures/` (`sli_dashboard`, calibration, wait vs queue, tier qualification).

## Live demo

- UI: https://demo.vgac.cloud/
- Deploy tree: private repo [espirado/vgac](https://github.com/espirado/vgac) — `scripts/deploy-demo.sh`, `infra/terraform/envs/demo`

VGAC tabs map to the poster: Predictions = QWR, Calibration = the gate, GPUs = GCE/HPH candidates, HPC = SAI candidate.

## Companion repositories

- [Reliability-First-Queue-Risk](https://github.com/espirado/Reliability-First-Queue-Risk) — Paper 2, QWR SLIs
- [Prediction-to-Policy-Integration](https://github.com/espirado/Prediction-to-Policy-Integration) — Paper 3 / VGAC policy
- `research-fall2025` — shared `src/sli` and `src/tier` (local data volume may be unmounted)

## Authors

- Andrew Espira, Saint Peter's University — [ORCID 0009-0002-9196-8094](https://orcid.org/0009-0002-9196-8094)
- Sharath Kumar, Saint Peter's University (companion papers; confirm Sessionize listing before printing the poster)

**Community context.** The framing — that AI GPU workloads lack an accepted golden-signal set — draws on public Google SRE community discussions on monitoring AI workloads in large production deployments.
