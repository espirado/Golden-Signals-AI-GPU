# Conversation Notes — Google SRE Discussion

**Date.** Weeks before this repo was created (spring/early summer 2026).
**Context.** Google SRE discussion on monitoring AI workloads in large production deployments. The specific talk cited: **"Monitoring AI workloads in large production deployments."**

**Speaker bios (for the acknowledgments section of the paper, pending approval).**

- **Google SRE contact** — Site Reliability Engineer at Google. Contributor to *The Site Reliability Workbook* (O'Reilly, 2018) and *Implementing Service Level Objectives* (O'Reilly, 2020). Research interests include distributed systems and machine learning.
- **Datadog contact** — Developer Advocate on Datadog's Advocacy team, previously at Google Cloud where he built and told stories about cloud infrastructure.

*Names to be added once each individual confirms they're comfortable being credited.*

## Framing the conversation established

1. **There is no accepted golden-signal set for AI workloads.** The classical four (latency, traffic, errors, saturation) travel poorly to training and inference on GPUs. Every large operator is inventing an incomplete equivalent, often internally, and there is no widely-cited canonical answer.
2. **Complexity is bimodal.** Both *training* (long-running, gradient-driven, data-parallel with collective ops) and *inference* (request/response but with heavy tail latency, KV-cache pressure, and quality signals distinct from HTTP status) each have their own failure modes that the classical golden signals cannot express.
3. **The hard question is "when to alert and when not to."** GPU cluster telemetry is high-volume and high-variance. Alerting on the wrong signal produces on-call fatigue; not alerting on the right signal produces silent quality regressions or capacity hogging.
4. **Cluster hogging.** A specific failure mode called out: one team's training job monopolizing a large fraction of a shared cluster, either intentionally or through mis-configured checkpointing / restart policies. The classical saturation metric does not distinguish "expected load" from "hogging."
5. **The signal set must be composable.** Whatever golden-signal set an SRE proposes for AI workloads needs to compose with existing SLO tooling (Prometheus / Datadog / OpenTelemetry / Google SRE burn-rate multi-window alerts) and existing SRE processes (runbooks, blameless postmortems, CUJ-driven SLIs).

## Open questions the conversation surfaced

- What is the AI-workload equivalent of the classical **CUJ** (Critical User Journey)?
- Should the golden signals be defined per-tenant, per-job-type, or per-cluster?
- How do we monitor **output quality** in a way that is cheap enough to run continuously in production?
- When is drift a golden signal on its own, and when is it a secondary signal derived from calibration / performance metrics?

## How this paper answers the framing

- Proposes **five golden signals** (see `docs/SIGNALS_TAXONOMY.md`) that map to the classical four but are defined *for* AI workloads:
  - Queue-of-Work Reliability (an SLI over calibrated wait-time probabilities) generalizes *latency*.
  - GPU-Compute Effectiveness generalizes *saturation* by replacing `util%` with a compound signal.
  - HBM-Pressure Headroom generalizes *saturation* on the memory dimension.
  - Straggler Amplification Index catches the collective-op-tail failure mode that classical *latency* misses.
  - Output-Quality Calibration generalizes *errors* to include silent quality regressions.
- Distinguishes **training** signals from **inference** signals where they diverge (Output-Quality Calibration has a different definition in each), while keeping the top four signals invariant.
- Provides **SLO templates** with burn-rate multi-window alerts (per `Implementing Service Level Objectives`) for each signal.
- Reports **empirical evidence** on four production and public traces, matching the standard the classical golden-signals literature set for its examples.

## What the paper does *not* claim

- That the classical four golden signals are wrong for general web services. They remain excellent for that case.
- That five is the "right" number. It is the *minimum viable set* our measurements support; ops teams may extend it.
- That the signals eliminate alert fatigue. They constrain what to alert on; alert-suppression policies, on-call runbooks, and postmortem discipline remain necessary.

## How to acknowledge in the final paper

Suggested language for the Acknowledgments section, subject to approval from both individuals before submission — names to be inserted once confirmed:

> We thank [Google SRE contact] and [Datadog contact] for a
> conversation that crystallized the framing of this work — that AI
> workloads on shared GPU clusters currently lack a widely-accepted
> golden-signal set, and that codifying one is a first-class SRE
> problem, not a monitoring-tool problem.

Any specific technical example or quote traced back to that conversation should be attributed inline in the paper text.
