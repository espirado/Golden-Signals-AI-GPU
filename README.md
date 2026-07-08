# Golden Signals for AI Workloads on Shared GPU Clusters

**Working title.** *Beyond the Four Golden Signals: An Empirical Study of GPU-Cluster Observability for AI Workloads.*

This repository is the shared working home for two artifacts on the same underlying research question:

| Artifact | Audience | Length | Venue candidates |
| --- | --- | --- | --- |
| **Research paper** (`tex/research/`) | ML systems / SRE researchers | 10–14 pp full paper | MLSys, SoCC, SREcon research track, NSDI operational, USENIX ATC |
| **Practitioner article** (`tex/practitioner/`) | SREs, ML infra operators | 3–6 pp extended abstract | SREcon (talk + short paper), USENIX `;login:`, ACM Queue |

The two artifacts share the same evidence base (`artifacts/`), the same taxonomy of proposed signals (`docs/SIGNALS_TAXONOMY.md`), and the same code (`code/`); the research paper adds measurement rigor and quantitative claims, and the practitioner article distills the operational playbook.

## The problem

The four **[golden signals of monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)** — *latency, traffic, errors, saturation* — were codified for stateless request/response services. They travel poorly to AI workloads on shared GPU clusters:

- **Latency** has at least three distinct meanings for a training or inference workload (token-level, batch-level, epoch-level) with different SLOs and none of them mapping to end-user wall-clock time in the classical sense.
- **Traffic** as QPS is meaningless; the equivalent is *tokens/s*, *samples/s*, or *active jobs*, each with different failure modes.
- **Errors** rarely surface as HTTP-style status codes. A silently miscalibrated model, a corrupted gradient reduction, or a job that produces a valid-looking but low-quality output all fail the classical "errors" definition.
- **Saturation** as GPU utilization is a well-known [lying metric](https://arthurchiao.art/blog/understanding-gpu-performance/): a card can be 100% "utilized" while spending most of its time on memory-bound operations, throttling on temperature, or waiting on NCCL stragglers. HBM saturation, compute-vs-memory-bound state, and inter-node collective-op tail latency are the operationally meaningful signals, and none of them are captured by `nvidia-smi util%`.

A conversation at Google SRE framed this as a first-class SRE problem: *there is currently no accepted golden-signal set for AI GPU workloads, and every large operator is inventing an incomplete one.*

## Our contribution

This repository proposes **five golden signals for AI workloads on shared GPU clusters** and validates them on measured data across four heterogeneous environments (Amazon EKS, AWS ParallelCluster Slurm HPC, Alibaba GPU trace v2020/2023, Google Borg 2019). The signals are:

1. **Queue-of-Work Reliability** — calibrated probability that a submitted job will exceed its wait SLO (ECE, tail-gap, PSI on submit-time features), extending the reliability-first framework of Espira & Kumar (2026).
2. **GPU-Compute Effectiveness** — a compound signal composed of DCGM temperature dispersion, utilization variance (CV, P90–P10), power-brake trip rate, and interconnect (NVLink / NCCL) tail latency, replacing `util%`.
3. **HBM-Pressure Headroom** — memory-used fraction *and* the derivative (fill rate), catching OOM cliffs before they page.
4. **Straggler Amplification Index** — the ratio of slowest-decile step time to median step time across data-parallel workers, i.e. the extent to which one degraded GPU is dragging the whole training or inference tier.
5. **Output-Quality Calibration** — for inference: rolling ECE / Brier on the model's own predictions against a labeled shadow stream; for training: gradient-norm and loss-curve anomaly detection against a per-model reference band.

Each signal ships with:

- a **precise definition** (equation + DCGM / scheduler feature it derives from),
- an **SLO template** (target, threshold, alert-suppression rule),
- **empirical evidence** on our four-environment dataset,
- and a **failure-mode taxonomy** (what each signal catches that the classical four miss).

## Repository layout

```
golden_signals_repo/
├── README.md                              this file (venue-agnostic roadmap)
├── LICENSE                                MIT
├── CITATION.cff                           machine-readable metadata
├── .zenodo.json                           Zenodo deposit metadata
├── docs/
│   ├── OUTLINE.md                         section-by-section outline for both artifacts
│   ├── SIGNALS_TAXONOMY.md                the five signals, defined and evidenced
│   ├── CRITIQUE.md                        why the classical four signals fail for AI
│   ├── SLO_TEMPLATES.md                   SLI/SLO/alert templates operators can copy
│   ├── CONVERSATION_NOTES.md              framing notes from the Google SRE community
│   └── RELATED_WORK.md                    SRE-book lineage; MLSys observability
├── tex/
│   ├── research/                          full research paper (draft in progress)
│   └── practitioner/                      SREcon / ;login: short piece (draft in progress)
├── code/                                  extractors, monitors, alert rules
├── artifacts/                             per-environment measurements
├── notebooks/                             the same reproducibility pattern as our other repos
└── figures/                               generated figures for both papers
```

## Relationship to prior work in this program

This paper stands on the shoulders of earlier work in the same GPU-cluster reliability program:

- **Espira & Kumar, ISS26** — *Reliability-First Queue Risk for GPU Clusters: Calibration, SLOs, and Reproducible Operational Integration.* Defines calibration monitoring as an SLI. This paper generalizes the SLI story from *queue risk* to a full golden-signal set.
- **Espira, Dhole, Kumar (TechRxiv 2026)** — *Calibration Under Extreme Imbalance: A Multi-Cluster Benchmark for Operational Queue-Delay Prediction.* Provides the cross-environment benchmark this paper reuses.
- **Espira & Kumar, PEARC '26** — *From Calibrated Probabilities to Scheduling Decisions* (VGAC). Provides the decision-layer treatment of the signals.

The Golden-Signals paper's novelty is (a) elevating the reliability-first framing from a single SLI to a full monitoring taxonomy that maps to Google SRE's four-golden-signal tradition, and (b) empirically validating each signal against ~1.2M pod events + 555 GPU Slurm jobs + two public traces.

## Data and reproducibility

The five signals are computed from data we have already collected and released with earlier papers:

- 1.2M EKS pod events (Kubernetes informer + cAdvisor + DCGM)
- 555 AWS ParallelCluster Slurm jobs (`sacct` + DCGM)
- Alibaba GPU trace v2020, v2023
- Google Borg 2019 trace

Sample slices (CSV, redistribution-safe) live under `artifacts/samples/`. Full data pointers are in `docs/DATA_PROVENANCE.md`.

## Status

Currently: scaffolded, outline drafted, related work list assembled. Next steps: complete signal-by-signal empirical evidence tables and draft the two LaTeX sources.

## Authors

- Andrew Espira, Saint Peter's University — [ORCID 0009-0002-9196-8094](https://orcid.org/0009-0002-9196-8094)
- Sharath Kumar, Saint Peter's University

**Community context.** The framing of this paper — that AI workloads on shared GPU clusters lack a widely-accepted golden-signal set and that SRE practice is currently inventing incomplete ones — draws on discussions in the Google SRE community around monitoring AI workloads in large production deployments.
