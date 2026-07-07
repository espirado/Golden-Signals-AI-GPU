# SLI/SLO Templates for the Five Golden Signals

These templates follow the *Implementing Service Level Objectives* (Hidalgo et al., O'Reilly 2020) convention:

- **SLI** — the measurable indicator.
- **SLO** — the target value or range for the SLI.
- **Error budget** — the complement of the SLO over a rolling window.
- **Alert** — a burn-rate multi-window rule that fires when the budget is being consumed too fast.

All examples below use Prometheus / PromQL notation for concreteness; the same recipes can be translated to Datadog metrics, OpenTelemetry meters, or a bespoke pipeline.

---

## Signal 1 — Queue-of-Work Reliability (QWR)

**SLI.** Rolling ECE of the queue-delay predictor over the last 7 days on labeled jobs.

```promql
queue_predictor_ece_7d{cluster=~"$cluster"}
```

**SLO.**
- Advisory tier: `queue_predictor_ece_7d <= 0.05`
- Gate tier: `queue_predictor_ece_7d <= 0.03`

**Burn-rate alert (two-window).**
```yaml
- alert: QWR_FastBurn
  expr: |
    (1 - sli:queue_predictor_calibrated:ratio_rate1h) > (14.4 * 0.05)
    and
    (1 - sli:queue_predictor_calibrated:ratio_rate5m) > (14.4 * 0.05)
  for: 2m
  labels: { severity: page }

- alert: QWR_SlowBurn
  expr: |
    (1 - sli:queue_predictor_calibrated:ratio_rate6h) > (6.0 * 0.05)
    and
    (1 - sli:queue_predictor_calibrated:ratio_rate30m) > (6.0 * 0.05)
  for: 15m
  labels: { severity: ticket }
```

**Suppression.** Skip alerting during recalibrator rollout windows tagged in the `recalibrator_rollout_seconds` metric.

---

## Signal 2 — GPU-Compute Effectiveness (GCE)

**SLI.** Fraction of the last N minutes classified as "useful compute" by the compound-signal classifier (see `code/monitors/gce.py`).

```promql
gce_useful_fraction:rate10m
```

**SLO.** `gce_useful_fraction:rate10m >= 0.85` over any 10-minute window.

**Burn-rate alert.**
```yaml
- alert: GCE_Degraded
  expr: |
    (1 - gce_useful_fraction:rate10m) > (14.4 * 0.15)
    and
    (1 - gce_useful_fraction:rate5m)  > (14.4 * 0.15)
  for: 2m
```

**Runbook pointer.** If GCE degrades, check in order: (1) thermal spread (`temp_p90 - temp_p10`), (2) power-brake trip count, (3) NCCL tail-latency histogram.

---

## Signal 3 — HBM-Pressure Headroom (HPH)

**SLI.** `1 - (mem_used / mem_total)` per node, aggregated as `min` across GPUs of interest.

```promql
min(hph:mem_headroom_ratio) by (job, gpu)
```

**SLO.** `hph:mem_headroom_ratio >= 0.10` at all times AND `oom_kills:sum1w == 0`.

**Alert.** A separate *derivative-based* rule so the alert fires *before* OOM:
```yaml
- alert: HPH_ImminentOOM
  expr: |
    min(hph:mem_headroom_ratio) < 0.10
    and
    deriv(hph:mem_headroom_ratio[5m]) < 0
  for: 1m
  labels: { severity: page }
```

**Suppression.** Ignore during checkpoint-restore windows.

---

## Signal 4 — Straggler Amplification Index (SAI)

**SLI.** Ratio of p90 to p50 step time across data-parallel workers of a job.

```promql
histogram_quantile(0.9, rate(step_time_bucket[10m]))
/
histogram_quantile(0.5, rate(step_time_bucket[10m]))
```

**SLO.** SAI $\leq 1.2$ over any 10-minute window.

**Alert.**
```yaml
- alert: SAI_StragglerAmplification
  expr: sai:ratio_10m > 1.5
  for: 20m
  labels: { severity: ticket }
```

**Runbook.** Identify slow worker via `job_worker:step_time:p50`, health-check the GPU (temperature, power, ECC), cordon, and drain the node.

---

## Signal 5 — Output-Quality Calibration (OQC)

### Inference variant

**SLI.** Rolling ECE against a labeled shadow stream.

```promql
oqc_inference_ece_7d{model=~"$model"}
```

**SLO.** `oqc_inference_ece_7d <= 0.05`.

**Alert.** Same burn-rate structure as QWR.

### Training variant

**SLI.** $z$-score of `(loss, grad_norm)` against a per-model healthy reference band.

```promql
oqc_training_zscore:max10m{run=~"$run"}
```

**SLO.** `oqc_training_zscore:max10m <= 3` over any 100-step window.

**Alert.**
```yaml
- alert: OQC_TrainingAnomaly
  expr: oqc_training_zscore:max10m > 3
  for: 10m
  labels: { severity: ticket }
```

---

## Composition: a single dashboard row per cluster

The five signals compose into one dashboard row with five sparklines and a *composite health index*:

$$
\text{ClusterHealth} \;=\; \min\bigl(1 - \text{QWR}/0.05,\; \text{GCE},\; \text{HPH},\; \tfrac{1}{\text{SAI}},\; 1 - \text{OQC}/0.05\bigr)
$$

If any of the five signals is out of budget, `ClusterHealth < 1`. This is the single number an on-call SRE should watch, with drill-down to the individual signals.
