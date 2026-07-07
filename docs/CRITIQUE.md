# Why the Classical Four Golden Signals Travel Poorly to AI Workloads on GPU Clusters

The four golden signals codified in the Google *SRE Book* — **latency, traffic, errors, saturation** — remain the correct default for stateless request/response services. This document argues, with concrete failure examples grounded in the four-environment dataset in this repository, that the four break down when they meet AI training and inference on shared GPU clusters, and that a substitute set is needed.

## Latency

**Classical definition.** Wall-clock time from a request entering the service to the response leaving it, measured at high percentiles (p50, p90, p99).

**Why it breaks.** An AI workload's latency has *at least three distinct meanings*:

| Latency variant | Where it matters | Why it is not the classical latency |
| --- | --- | --- |
| **Token-level** ($t_{\text{token}}$) | LLM inference streaming | The "request" doesn't complete on the first response byte; it streams. p99 of first-token latency is only half the story; token-to-token tail also matters. |
| **Batch-level** ($t_{\text{batch}}$) | Recommendation / vision inference | A batch of $N$ requests sees one latency; classical per-request latency is derived from batching policy, not from load. |
| **Epoch-level** ($t_{\text{epoch}}$) | Training | Wall-clock time per epoch or per step. The user is the ML engineer, not the end-user. |
| **Queue-level** ($t_{\text{queue}}$) | Any workload on a shared cluster | Time from `sbatch` / `kubectl apply` to first CUDA context. Often *larger than* all three above and invisible to service-level latency probes. |

**Empirical evidence.** On the 555-job Slurm trace, wall-clock queue latency (Slurm submit-to-start) is uncorrelated with service-level latency ($r < 0.1$). Alerting on classical p99 latency would miss queue-based user-visible slowness entirely.

## Traffic

**Classical definition.** Requests per second (or bytes per second, or transactions per second).

**Why it breaks.**

- The equivalent for **inference** is *tokens/s* or *samples/s*, not *req/s*. These are decoupled: 1 QPS at 4096 tokens per response is a very different load than 100 QPS at 40 tokens per response.
- The equivalent for **training** is *active jobs on the cluster* — a bounded, low-cardinality quantity that a QPS-style monitor is not designed for.
- **Traffic is bimodal.** Training and inference workloads on the same cluster have opposite temporal profiles: training is long-lived, gradient-driven, checkpoint-heavy; inference is short-lived, request-driven, memory-cache-heavy. Aggregating them into "cluster traffic" hides both.

## Errors

**Classical definition.** Rate of requests returning non-2xx (HTTP) or protocol-specific error codes, sometimes weighted by severity.

**Why it breaks.**

- The classical error rate misses **silent quality regressions**. A model that returns a valid 200 response but a hallucinated / miscalibrated / low-quality output is failing but is invisible.
- Training-side "errors" are a different beast entirely — a gradient-explosion NaN is an error; a slowly diverging loss is a *quality regression* that no `except:` block catches.
- **OOM kills** and **HW_SLOWDOWN throttling** manifest as job restarts or extended step times, not as request errors. They should be first-class signals.

**Empirical evidence.** On the EKS 1.2M-event trace, request-level HTTP errors correlate weakly ($r \approx 0.2$) with GPU-side reliability signals such as ECE drift and thermal throttling episodes. The two channels are largely independent — meaning error-rate alerting alone leaves a substantial class of AI failures dark.

## Saturation

**Classical definition.** How full the service is (queue length, CPU load, memory pressure), typically as a fraction of capacity.

**Why it breaks — and why this is the most damaging of the four.** The default AI-cluster proxy is `nvidia-smi util%`. That metric is a *lying signal*:

1. `util%` is a *time-based* metric (fraction of time the GPU has any kernel active), not a *throughput-based* metric. A kernel that reads memory the whole time and computes very little reports `util% = 100`.
2. **Memory-bound** ops on modern GPUs (LLM inference decode, gradient reduction) commonly show `util% = 100` while doing very little compute. Compute-bound and memory-bound utilization deserve different alerts.
3. **Thermal throttling** on A100 ≥ 85°C and H100 ≥ 87°C reduces effective throughput without changing `util%`.
4. **Power-brake trips** (HW_POWER_BRAKE) on badly-provisioned racks reduce effective throughput while `util%` stays high.
5. **Interconnect (NVLink / NCCL) tail latency** — one slow card in an all-reduce — is invisible in single-GPU `util%`.
6. **HBM pressure** — the other saturation dimension — is separately reported by DCGM and is a distinct signal.

**Empirical evidence.** In the DCGM Slurm slice, `util%` is uncorrelated with useful goodput ($r \approx 0.1$) once thermal and power-brake episodes are conditioned on. Signals 2 (GCE), 3 (HPH), and 4 (SAI) between them capture the four distinct GPU saturation modes classical `util%` conflates.

## Summary

| Classical signal | Failure mode on AI/GPU workloads | Proposed replacement(s) |
| --- | --- | --- |
| Latency | Three or four distinct latencies; per-request p99 misses queue and token variants | QWR (queue) + latent token-level metric (out of scope) |
| Traffic | QPS is meaningless; bimodal train + inference load | Domain-specific traffic definition per workload class (out of scope of the *golden* set) |
| Errors | HTTP-style errors miss silent quality regressions | OQC (inference and training variants) |
| Saturation | `util%` is a lying signal; four distinct saturation modes are conflated | GCE + HPH + SAI (three signals from one) |

The claim of this paper is not that latency, traffic, errors, and saturation are wrong concepts — they are the right *categories*. The claim is that each category needs a *precise operational definition* for AI workloads, and that the *golden* subset appropriate to a paging on-call rotation is the five signals proposed here.
