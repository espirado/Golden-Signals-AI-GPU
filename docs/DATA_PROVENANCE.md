# Data Provenance and Reuse Notes

QWR reuses queue-delay traces from earlier papers in this program.
GCE, HPH, SAI, and OQC are defined here; they are not yet extracted
from those traces.

## Environments we can point at

| Environment | Scale | What it actually supports | Where it lives |
| --- | --- | --- | --- |
| AWS ParallelCluster Slurm | 555 jobs, 38 features | **QWR.** Wait labels and queue features are real. DCGM util/mem/power columns are idle (T4 queue-wait experiment). | `Reliability-First-Queue-Risk/data/training_dataset.csv` |
| Amazon EKS | Live model card: 5,184 jobs; Paper 3 abstract: 650 job records; narrative also cites ~1.2M feature rows | **QWR** on the VGAC production model. Full 1.2M rows are on an unmounted volume (`research-fall2025/data` → `/Volumes/data-esp/...`). | [demo.vgac.cloud](https://demo.vgac.cloud/) `/model/health`; Paper 3 tex |
| Alibaba GPU traces | Public | Queue-style features in Paper 1 / paper4 ETL. Not GCE/SAI. | `research-fall2025` when the data volume is mounted |
| Google Borg 2019 | Public | Same as Alibaba — scheduler transfer, not GPU effectiveness. | same |

## Per-signal status

- **QWR** — computed in Paper 2 (`src/sli/compute.py`, `notebooks/train_models.py`). Live inference is VGAC `/api/predict/wait`.
- **GCE / HPH** — collector exists (`research-fall2025/scripts/dcgm_job_stats.py`). Do not compute them from the 555-job CSV.
- **SAI** — needs per-worker step times. Not in any current tree.
- **OQC** — would be ECE of *model outputs*, not of the queue-delay classifier.

## Redistribution

- Alibaba and Borg: public.
- EKS production identifiers were hashed at collection. Full trace stays on private infrastructure.
- Slurm 555: Saint Peter's testbed; ships in Reliability-First-Queue-Risk.
