# Related Work — Reading List and Positioning Notes

## SRE lineage (essential context)

- **B. Beyer et al.**, *Site Reliability Engineering* (O'Reilly, 2016). Chapter 6 codifies the four golden signals. This paper's contribution is best framed as *"the AI-workload equivalent of Chapter 6"*.
- **B. Beyer, N. Murphy et al.**, *The Site Reliability Workbook* (O'Reilly, 2018). Salim Virji is a contributor; useful for practical CUJ / SLI worked examples.
- **A. Hidalgo et al.**, *Implementing Service Level Objectives* (O'Reilly, 2020). Salim Virji is also a contributor. Multi-window burn-rate alerts and error-budget policies. All SLO templates in `docs/SLO_TEMPLATES.md` follow this book's conventions.
- **Rothenberg & Chevalier**, *Learning Multi-window Burn Rate Alerts.* Google SRE blog / whitepaper.

## GPU cluster telemetry (measurement priors)

- **Meta AI**, *The Llama 3 Herd of Models* (arXiv:2407.21783, 2024). Failure taxonomy across 16k-GPU H100 training; HW_SLOWDOWN and HBM-OOM as leading causes.
- **J. Belotti, A. Chang**, *Keeping 20,000 GPUs Healthy* (Modal blog, Dec 2025). Passive-healthcheck taxonomy; source for many DCGM dispersion features used in this paper.
- **P. Deutsch et al.**, *PinDrop: Breaking the Silence on SDCs in a Large-Scale Fleet* (IEEE HPCA 2026). Silent Data Corruption episodes correlate with memory-stress features; the primary evidence for the OQC-inference signal and the SAI signal.

## ML observability tools and academic work

- **MLPerf**, *Training* and *Inference* benchmark suites — establish per-workload cost accounting but not runtime observability signals.
- **Fiddler AI**, **Arize AI**, **WhyLabs** — commercial ML observability tools; useful surveys of what practitioners already deploy. Positioning: this paper proposes the *underlying signals* those tools should surface.
- **Datadog LLM Observability**, **New Relic AI Monitoring** — production monitoring tools with AI/LLM-specific views; useful as a comparison for coverage.
- **Google Vertex AI Model Monitoring** — drift detection reference implementation.

## Calibration and drift (technical basis)

- **C. Guo, G. Pleiss, Y. Sun, K. Q. Weinberger**, *On Calibration of Modern Neural Networks*, ICML 2017. Foundation of Signal 5 (OQC).
- **B. Zadrozny, C. Elkan**, *Transforming Classifier Scores into Accurate Multiclass Probability Estimates*, KDD 2002. Isotonic calibration.
- **N. Siddiqi**, *Credit Risk Scorecards*, Wiley 2005. Source of the PSI thresholds ($0.1$ moderate, $0.25$ severe).

## GPU cluster scheduling (context for QWR)

- **A. Verma et al.**, *Large-Scale Cluster Management at Google with Borg*, EuroSys 2015.
- **I. Gog et al.**, *Firmament: Fast, Centralized Cluster Scheduling at Scale*, OSDI 2016.
- **H. Mao et al.**, *Learning Scheduling Algorithms for Data Processing Clusters*, SIGCOMM 2019.

## Traces we use

- **Q. Weng et al.**, *MLaaS in the Wild*, NSDI 2022. Alibaba GPU trace v2020.
- **Q. Weng et al.**, *Beware of Fragmentation*, ATC 2023. Alibaba GPU trace v2023.
- Google *Borg 2019* trace.

## Own prior work (this program)

- **A. Espira and S. Kumar**, *Reliability-First Queue Risk for GPU Clusters: Calibration, SLOs, and Reproducible Operational Integration.* ISS26, 2026. Defines calibration monitoring as an SLI; this paper generalizes the SLI story to five signals.
- **A. Espira, T. Dhole, S. Kumar**, *Calibration Under Extreme Imbalance: A Multi-Cluster Benchmark for Operational Queue-Delay Prediction.* TechRxiv 2026, doi:10.36227/techrxiv.177041829.96464119.
- **A. Espira and S. Kumar**, *From Calibrated Probabilities to Scheduling Decisions: Decision Rules and Calibration Prerequisites for GPU Queue Policy.* PEARC '26 (VGAC). Decision-layer treatment.

## Positioning against three closest works

1. *SRE Book Chapter 6* — canonical golden signals. We propose the AI-workload extension.
2. *Meta LLaMA-3 paper (Section on infrastructure failures)* — describes what fails, not how to monitor for it. We convert their failure taxonomy into monitored signals with SLOs.
3. *Modal healthchecking* — describes active healthchecks that take ~1h and require exclusive node access. We show that the same failure modes are visible in *passive* DCGM aggregates at zero scheduling cost.

## Reading list still-to-consult

- **Datadog's DASH 2026 talks** on AI observability (once available).
- **Google Cloud AI Hypercomputer observability whitepaper** (if a public version exists).
- **AWS Trainium / Inferentia monitoring** — comparison point for non-NVIDIA silicon; may broaden or narrow the paper's applicability claims.
