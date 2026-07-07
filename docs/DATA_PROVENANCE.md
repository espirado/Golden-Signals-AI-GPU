# Data Provenance and Reuse Notes

This paper does not require new data collection. It reuses evidence from
four environments already gathered and released in prior work in this
research program, plus per-signal extractions that are computed on-the-fly
from those existing traces.

## Environments

| Environment | Scale | Source | Where it lives in this workspace |
| --- | --- | --- | --- |
| Amazon EKS (K8s) | 1.2M pod events | Kubernetes informer + cAdvisor + DCGM exporter (production cluster; identifiers hashed) | `iss26_work/paper2_repo/data/samples/eks_dec_sample.csv` (redistribution-safe slice) |
| AWS ParallelCluster Slurm | 555 GPU jobs, 38 features | `sacct` + per-node DCGM snapshots at submit timestamps | `iss26_work/paper2_repo/data/samples/slurm_training_dataset.csv` |
| Alibaba GPU trace v2020, v2023 | Public | GitHub trace-replay pipeline; pending-ratio reconstructed | `iss26_work/paper2_repo/data/samples/alibaba_sample.csv` |
| Google Borg 2019 | Public | ClusterData BigQuery release | `iss26_work/paper2_repo/data/samples/borg_sample.csv` |

## Per-signal extraction recipes (planned locations)

- `code/extract_qwr.py` — reads Slurm and EKS traces, computes QWR = rolling ECE on the wait-time predictor. Depends on the calibrated predictor released in the ISS26 companion.
- `code/extract_gce.py` — reads DCGM per-node aggregates, computes GCE via the small logistic-regression composition described in `docs/SIGNALS_TAXONOMY.md`.
- `code/extract_hph.py` — reads DCGM memory aggregates, computes HPH and its first derivative.
- `code/extract_sai.py` — reads per-worker step-time telemetry. Only Slurm and EKS have this; Alibaba and Borg traces lack per-worker granularity.
- `code/extract_oqc.py` — reads model predictions and labels (shadow stream). Only synthetically constructed for the Slurm trace; a live inference-side OQC measurement is a follow-on.

## Redistribution status

- **Alibaba** — public, redistributable as-is.
- **Google Borg 2019** — public via ClusterData BQ release, redistributable.
- **EKS 1.2M** — production trace; identifiers hashed at collection time; a redistribution-safe slice (`n ~ 500`) ships in the ISS26 repo. Full trace stays on private Saint Peter's AWS infrastructure.
- **AWS ParallelCluster Slurm 555** — collected on Saint Peter's testbed, no external identifiers; full trace ships in the ISS26 repo.

## Ethical and privacy notes

The EKS production trace was collected via the Kubernetes informer API on a cluster the authors administer. Pod identifiers, namespaces, and user labels were hashed before any data left the cluster. No personal data of end users appears in any release.
