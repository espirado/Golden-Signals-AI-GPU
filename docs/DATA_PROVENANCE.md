# Data Provenance and Reuse Notes

QWR reuses queue-delay traces from earlier papers in this program.
GCE, HPH, SAI, and OQC are defined here; they are not yet extracted
from those traces.

## Environments we can point at

| Environment | Scale | What it actually supports | Where it lives |
| --- | --- | --- | --- |
| AWS ParallelCluster Slurm | 555 jobs | **QWR, held-out.** Print `artifacts/slurm_real_benchmark.json` (444/111, GB AUROC 0.863 CI 0.71–0.96, ECE 0.056, suggest). Label = wait > P90 133s (~10% base rate). DCGM util/mem/power columns are idle. | Public: [Reliability-First-Queue-Risk](https://github.com/espirado/Reliability-First-Queue-Risk) `artifacts/` + `data/samples/`. Local-only: `results/model_evaluation_slurm.json` (0.985/0.006 on 388/167) — do not cite. |
| Amazon EKS | Live card n=5,184; Paper 3 abstract n=650; some plots say 4,000; narrative also cites ~1.2M feature rows | **QWR** on the VGAC production model (ECE 0.068, **warn** vs 0.05/0.03). Label = wait > 120s (near median). | [demo.vgac.cloud](https://demo.vgac.cloud/) `/model/health`; Paper 3 tex. Full 1.2M rows are on an unmounted volume. |
| Alibaba GPU traces | Public | Queue-style features in Paper 1 / paper4 ETL. Not GCE/SAI. | `research-fall2025` when the data volume is mounted |
| Google Borg 2019 | Public | Same as Alibaba — scheduler transfer, not GPU effectiveness. | same |

## Per-signal status

- **QWR** — public held-out numbers in `artifacts/slurm_real_benchmark.json`. Live inference is VGAC `/api/predict/wait`. Do not use `data/samples/drift_metrics.json` (ECE 0 / AUROC 1 in 7 of 8 windows; PSI ≈ 12.41 is empty-bin).
- **GCE / HPH** — collector exists (`research-fall2025/scripts/dcgm_job_stats.py`). Do not compute them from the 555-job CSV.
- **SAI** — needs per-worker step times. Not in any current tree.
- **OQC** — would be ECE of *model outputs*, not of the queue-delay classifier.

## Redistribution

- Alibaba and Borg: public.
- EKS production identifiers were hashed at collection. Full trace stays on private infrastructure.
- Slurm 555: Saint Peter's testbed. Public companion ships samples + artifacts, not the full `training_dataset.csv`.
