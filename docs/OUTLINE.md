# Section-by-section outlines for the two artifacts

Both drafts share a single evidence base. Where a section number below is prefixed with **R** it appears in the research paper; **P** means the practitioner article.

## Research paper (`tex/research/`) — target 10–14 pp

**R1. Introduction.** Golden signals travel poorly to AI GPU workloads. Motivating example: an EKS cluster with 100% `util%` and healthy p99 request latency is silently dropping 22% goodput to stragglers. Contribution list: five signals + empirical validation + SLO templates + operator runbook.

**R2. Background.**
- R2.1 Classical four golden signals (Beyer et al., *SRE Book* ch.6) and their assumptions (stateless service, request/response).
- R2.2 Why those assumptions break for GPU AI workloads (long-lived jobs, collective ops, memory-bound compute, quality as a signal).
- R2.3 Prior work on ML observability (references only; the field is fragmented).

**R3. Method: five signals defined precisely.**
- R3.1 QWR — calibrated queue-of-work reliability.
- R3.2 GCE — GPU-compute effectiveness (compound signal, replacing `util%`).
- R3.3 HPH — HBM-pressure headroom.
- R3.4 SAI — straggler amplification index.
- R3.5 OQC — output-quality calibration (train and inference variants).
- R3.6 Composition: the five as a monitoring **basis**, not five isolated alerts.

**R4. Data and measurement protocol.**
- Amazon EKS (1.2M pod events)
- AWS ParallelCluster Slurm (555 jobs + DCGM)
- Alibaba v2020, v2023
- Google Borg 2019
- Per-signal extraction recipe cross-referenced to `code/`.

**R5. Empirical results.**
- R5.1 Each signal's behavior across the four environments.
- R5.2 The cross-signal correlation table from `SIGNALS_TAXONOMY.md` — argument that the five are mutually informative, not redundant.
- R5.3 Failure-mode case studies (e.g., "GCE catches the throttling episode `util%` marks green").
- R5.4 SLO burn-rate simulation: given a workload trace, how the alerts of §III fire vs. classical golden-signal alerts.

**R6. Operational integration.**
- R6.1 A monitoring framework tying the five signals to existing SRE tooling (Prometheus, OpenTelemetry, Datadog).
- R6.2 Tie-in with the tier-qualification framework from Espira & Kumar (2026) — the five signals are the SLIs; the four tiers are the policy layer.
- R6.3 A calibration-monitor as a shared sub-component (three of five signals reduce to a calibration measurement).

**R7. Discussion and limitations.** Small Slurm trace, honest cross-cluster transfer limits, the training-side OQC variant needs a live measurement study.

**R8. Related work.**
- SRE lineage: *SRE Book*, *Site Reliability Workbook*, *Implementing SLOs*.
- ML observability tools: MLPerf, DCGM, Fiddler, Arize, Datadog MLOps.
- Recent GPU-cluster reliability: Meta LLaMA-3, Modal, PinDrop.
- Calibration: Guo et al. 2017 and downstream work.

**R9. Conclusion.**

**R10. AI-usage disclosure + Acknowledgments** (crediting Salim Virji and Max Saltonstall for the framing conversation, subject to their approval).

## Practitioner article (`tex/practitioner/`) — target 3–6 pp

**P1. Motivation.** Two-paragraph version of R1.

**P2. Why the classical four golden signals travel poorly.** Compressed R2.

**P3. Five signals you can start monitoring today.**
- Same content as R3, but each signal capped at one page.
- Each signal includes a *"what to alert on"* box and a *"what NOT to alert on"* box.

**P4. Copy-pasteable SLO templates.** Prometheus / OpenTelemetry snippets from `docs/SLO_TEMPLATES.md`.

**P5. A short case study from the four-environment dataset.**

**P6. What we don't know yet.** Honest open-question list — matches R7.

**P7. Acknowledgments and further reading.**

## Positioning statement (the elevator pitch, for both artifacts)

*The four golden signals — latency, traffic, errors, saturation — were codified for stateless request/response services. AI workloads on GPU clusters are neither stateless nor request/response, and every large operator has been forced to invent an incomplete set of substitute signals. We propose five: Queue-of-Work Reliability, GPU-Compute Effectiveness, HBM-Pressure Headroom, Straggler Amplification Index, and Output-Quality Calibration. We define each precisely, provide an SLO template with burn-rate multi-window alerts, and validate them on ~1.2M pod events across four heterogeneous environments. The signals are cheap to compute from telemetry a well-instrumented cluster already emits, and they compose with existing SRE tooling.*
