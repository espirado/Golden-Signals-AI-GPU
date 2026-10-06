# The Five Golden Signals — Taxonomy

For AI workloads on shared GPU clusters, we propose five golden signals. Each is designed to (a) be **cheaply computable** from telemetry already present on a well-instrumented cluster (DCGM + scheduler + serving-layer), (b) admit a **precise SLO definition** with a target and a burn-rate alert, and (c) map to a **failure mode** that the classical four golden signals miss.

Notation. $\hat{p}$ = a calibrated predicted probability; ECE = Expected Calibration Error; PSI = Population Stability Index; CV = coefficient of variation; P$k$ = $k$-th percentile.

---

## Signal 1 — Queue-of-Work Reliability (QWR)

**Generalizes.** Latency.

**Definition.** At each submit timestamp, a queue-delay predictor produces $\hat{p} = P(\text{wait} > \text{SLO}_{\text{wait}})$. QWR is the rolling calibration error of that predictor:

$$
\text{QWR}_w \;=\; \text{ECE}_{15}(\hat{p}_t, y_t \mid t \in W)
$$

on a window $W$ (default 7 days for production, 250 jobs for HPC). Complementary signals: tail-gap at $\tau=0.7$, PSI on `pending_ratio` between window and deployment reference.

**SLO template.**
- Target: QWR $\leq 0.05$ (advisory tier); QWR $\leq 0.03$ (gate tier).
- Alert (burn-rate, two-window): $2\%$ error budget / $1$ h AND $5\%$ budget / $6$ h $\Rightarrow$ page; single-window at $5\%/24\text{h}$ $\Rightarrow$ ticket.
- Suppression: hold alert during known-drift windows tagged by the recalibrator's rollout log.

**What it catches that classical latency misses.**
- A cluster whose *mean* wait is fine but whose *tail* is silently degrading.
- A predictor that has become miscalibrated (a "50%" wait risk is really 90%) without the underlying latency distribution changing.
- Cross-cluster transfer degradation — new cluster with familiar-looking metrics but very different queue dynamics.

**Evidence.** Held-out Slurm GB in the public companion: AUROC 0.863 (CI 0.71–0.96), ECE 0.056, n_test=111, tier suggest — `Reliability-First-Queue-Risk/artifacts/slurm_real_benchmark.json`. Live VGAC `/model/health`: ECE 0.068 on n=5,184 (warn vs 0.05 advisory / 0.03 gate). Do not cite local-only `results/model_evaluation_slurm.json` (0.985 / 0.006). Long-wait labels differ: Slurm > P90 133s (~10%); EKS > 120s (near median).

---

## Signal 2 — GPU-Compute Effectiveness (GCE)

**Generalizes.** Saturation, on the compute dimension.

**Definition.** GCE is a *compound* signal derived from DCGM per-node aggregates over a rolling window:

$$
\text{GCE} \;=\; f(\underbrace{\overline{\text{util}}}_{\text{mean util}},\; \underbrace{\text{CV}(\text{util})}_{\text{dispersion}},\; \underbrace{P90(\text{temp}) - P10(\text{temp})}_{\text{thermal spread}},\; \underbrace{\text{power-brake rate}}_{\text{HW\_POWER\_BRAKE trips / min}})
$$

The composition $f$ is a small learned classifier (logistic regression is sufficient) trained to distinguish "useful compute" from "throttling / stalling / straggler" states, with `util%` as one feature among four. `util%` alone is not a golden signal in AI clusters — it is one feature of one.

**SLO template.**
- Target: GCE $\geq 0.85$ (useful-compute fraction of the window).
- Alert: two-window burn-rate on the useful-compute error budget.

**What it catches that classical saturation misses.**
- The **lying-utilization** failure mode: `util%` = 100 while the card is memory-bound (compute idle, memory bus saturated).
- **Straggler warming**: a single node running hot and about to throttle, before it actually trips HW_SLOWDOWN.
- **Power-brake trips** which correlate with poorly-provisioned racks and produce mysterious throughput drops.

**Evidence source.** Failure modes cited in Meta LLaMA-3 [1], Modal healthchecking [2], and PinDrop [3]. The DCGM slice in this program is idle T4 queue-wait — do not compute GCE from it. There is no `code/monitors/gce.py` in this repo.

---

## Signal 3 — HBM-Pressure Headroom (HPH)

**Generalizes.** Saturation, on the memory dimension.

**Definition.** HPH is the fraction of HBM headroom remaining, *plus* its first derivative:

$$
\text{HPH}(t) \;=\; 1 - \frac{\text{mem\_used}(t)}{\text{mem\_total}(t)}, \quad
\dot{\text{HPH}}(t) \;=\; \frac{d}{dt}\text{HPH}(t)
$$

An alert fires when $\text{HPH} < 0.1$ **and** $\dot{\text{HPH}} < 0$ over a short window — i.e., the card is close to full *and* still filling.

**SLO template.**
- Target: $\text{HPH} \geq 0.10$ at all times AND OOM-kill count $= 0$ per week.
- Alert: derivative-based; suppresses alarms during known checkpoint-restore windows.

**What it catches that classical saturation misses.**
- HBM OOM cliffs on H100-class hardware, where 80 GB fills fast under sharded training.
- Silent KV-cache eviction on LLM inference before it manifests as latency.

**Evidence source.** DCGM `mem_used_mb_max` per-cluster time series; cited in [1] (Meta LLaMA-3 as a top-3 failure class).

---

## Signal 4 — Straggler Amplification Index (SAI)

**Generalizes.** Latency, on the collective-operation dimension.

**Definition.** For a data-parallel training job or a tensor-parallel inference tier, SAI is the ratio of the slowest-decile step time to the median step time across workers:

$$
\text{SAI} \;=\; \frac{P90(\text{step\_time}_i)}{P50(\text{step\_time}_i)},\quad i \in \text{workers}
$$

$\text{SAI} \approx 1.0$ is healthy; $\text{SAI} > 1.5$ means one slow GPU is dragging the whole tier. The shortcut “useful goodput $= 1/\text{SAI}$” assumes **fully synchronous** data-parallel steps; it does not hold for pipeline/async schedules.

**SLO template.**
- Target: SAI $\leq 1.2$ over any 10-minute window.
- Alert: crossing $1.5$ for two consecutive windows.
- Runbook: identify the slow worker (from per-GPU step-time telemetry), health-check it, cordon and drain.

**What it catches that classical latency misses.**
- The single-degraded-GPU failure mode PinDrop specifically documents [3]: NCCL tail latency from one slow card in an all-reduce, dropping goodput by 20–40%.
- Silent thermal throttling on one card among many.

**Evidence source.** Candidate only. Existing DCGM is idle; there are no per-worker step times in any current tree. PinDrop [3] is the literature prior, not a measurement we ran.

---

## Signal 5 — Output-Quality Calibration (OQC)

**Generalizes.** Errors — but for AI workloads, "error" is not an HTTP status code.

**Definition.** Two variants:

- **Inference.** Rolling ECE/Brier of the deployed model against a labeled shadow stream (either a golden-set replay or a low-rate live-labeling channel).
- **Training.** Anomaly score of `(loss, grad_norm)` vs. a per-model reference band captured on a healthy run.

$$
\text{OQC}_\text{inf} \;=\; \text{ECE}_{15}(\hat{p}_t, y_t \mid t \in W_{\text{shadow}}), \qquad
\text{OQC}_\text{train} \;=\; z\bigl((\ell_t, \|\nabla_t\|); \text{ref band}\bigr)
$$

**SLO template.**
- Inference: OQC $\leq 0.05$ over 7-day window.
- Training: OQC $z$-score $\leq 3$ over any 100-step window.

**What it catches that classical errors miss.**
- Silent quality regressions from data drift, model drift, or an upstream corruption of the feature store.
- Poisoned inputs or accidental label leakage in a shadow-labeling pipeline.
- Gradient-explosion or loss-plateau conditions that are not crashes but are wasted compute.

**Evidence source.** Extends the reliability-first framework from ISS26 to inference-time monitoring. Training-side reference bands are new for this paper.

---

## Cross-signal correlation matrix (to be populated with measurements)

| | QWR | GCE | HPH | SAI | OQC |
| --- | --- | --- | --- | --- | --- |
| **QWR** — Queue reliability | 1.0 | | | | |
| **GCE** — Compute effectiveness | *low* | 1.0 | | | |
| **HPH** — Memory headroom | *low* | medium | 1.0 | | |
| **SAI** — Straggler amp. | medium | high | low | 1.0 | |
| **OQC** — Output quality | low | low | medium | low | 1.0 |

This matrix is **empty on purpose**. Do not draft §V as if the five signals were validated on four environments. Only QWR has numbers.

---

## Citations (placeholder — will move to `tex/refs.bib`)

[1] Meta AI. *The Llama 3 Herd of Models.* arXiv:2407.21783 (2024).
[2] Belotti, J. and Chang, A. *Keeping 20,000 GPUs Healthy.* Modal Blog (Dec 2025).
[3] Deutsch, P. W. *et al.* *PinDrop: Breaking the Silence on SDCs in a Large-Scale Fleet.* IEEE HPCA 2026.
[4] Beyer, B. *et al. Site Reliability Engineering.* O'Reilly, 2016 — Chapter 6, "Monitoring Distributed Systems" (the four golden signals).
[5] Alex Hidalgo *et al. Implementing Service Level Objectives.* O'Reilly, 2020 — burn-rate multi-window alerting.
