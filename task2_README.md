# CS 683 PA2 — Task 2: Instruction-Aware L2 Replacement

This README section documents the current implementation, experiments, and measured results for **Task 2A and Task 2B** of Programming Assignment 2.

> **Status:** Task 2A experiments are complete. Task 2B policy/threshold experiments are complete for IPC, including the score-only comparison. The final Task 2B MPKI table/plots still need to be extracted from the existing logs. Task 3 is separate and is **not claimed as completed or validated here**.

## 1. Experimental setup

- **Simulator:** ChampSim
- **Warmup:** 25 million instructions per trace
- **Measurement:** 25 million instructions per trace
- **Traces:**
  - `traces/Trace1-003_.gz`
  - `traces/Trace2-002_.gz`
  - `traces/Trace3-001_.gz`
- **Baseline replacement policy:** LRU
- **Main metrics:** IPC, speedup over LRU, total L2 MPKI, load MPKI, data-load MPKI, and instruction-load MPKI

Speedup is calculated as:

\[
\text{Speedup} = \frac{\text{IPC of tested policy}}{\text{IPC of LRU baseline}}
\]

All performance numbers below are from completed simulations. Do not treat a policy as universally better based on one trace: results vary by trace.

## 2. Task 2A — Instruction-aware L2 replacement

### Policy: `irrip_mild`

The implementation is in `replacement/irrip_mild.l2c_repl`.

The policy uses different insertion priorities for instruction and data lines:

- Instruction line insertion RRPV: **1**
- Data line insertion RRPV: **2**
- On a hit, promote the line to RRPV **0**
- When possible, prefer evicting a non-instruction line already at RRPV 3
- Age non-instruction lines first, then use standard RRIP aging as a fallback

The intent is to give instruction lines stronger protection while retaining a mechanism to evict lines that appear less useful.

### Measured results

| Trace | LRU IPC | `irrip_mild` IPC | Speedup | Total L2 MPKI | Load MPKI | Data-load MPKI | Instruction-load MPKI |
|---|---:|---:|---:|---:|---:|---:|---:|
| Trace 1 | 0.232288 | 0.231425 | 0.9963× | 31.4867 | 19.0334 | 10.0096 | 9.02384 |
| Trace 2 | 0.250638 | 0.260640 | 1.0399× | 34.8653 | 20.9342 | 12.3517 | 8.58244 |
| Trace 3 | 0.382008 | 0.384235 | 1.0058× | 8.7108 | 3.92644 | 3.27304 | 0.65340 |

### Interpretation

- Trace 1 IPC is slightly lower than LRU (about **0.37% lower**).
- Trace 2 IPC improves by about **3.99%**.
- Trace 3 IPC improves by about **0.58%**.
- Therefore, `irrip_mild` is a useful Task 2A candidate, but it does not improve IPC on every trace.

Result logs:

- `results/full_Trace1-003__irrip_mild_w25m_s25m.log`
- `results/full_Trace2-002__irrip_mild_w25m_s25m.log`
- `results/full_Trace3-001__irrip_mild_w25m_s25m.log`

## 3. Task 2B — Dynamic set-dueling RRIP / I-RRIP

Task 2B explores dynamically selecting between RRIP and instruction-aware RRIP using leader sets and a bounded decision counter.

### Dynamic weighted policy

The weighted implementation uses leader sets selected by set index:

- `set % 64 == 0`: RRIP leader sets
- `set % 64 == 1`: I-RRIP leader sets

The policy tracks instruction and non-instruction accesses/misses and uses a weighted score to compare the policies. The instruction-miss weight and decision threshold were varied in experiments. The counter uses a window of **32,768** updates in the experiments summarized below.

The weight-4 variants are named `dyn_irrip_weighted_w4_t10`, `dyn_irrip_weighted_w4_t20`, etc. The suffix `t20`, for example, denotes the tested decision threshold configuration; it should not be interpreted as a percentage unless the implementation explicitly defines it that way.

### IPC threshold sweep

| Configuration | Trace 1 IPC | Trace 2 IPC | Trace 3 IPC |
|---|---:|---:|---:|
| `dyn_irrip_weighted_w4_t10` | 0.232327 | 0.260311 | 0.382292 |
| `dyn_irrip_weighted_w4_t20` | 0.232327 | 0.260375 | 0.382553 |
| `dyn_irrip_weighted_w4_t25` | 0.232327 | 0.260375 | 0.382334 |
| `dyn_irrip_weighted_w4_t30` | 0.232327 | 0.260375 | 0.381576 |
| `dyn_irrip_weighted_w4_t40` | 0.232327 | 0.256815 | 0.381661 |
| **LRU baseline** | **0.232288** | **0.250638** | **0.382008** |

For `dyn_irrip_weighted_w4_t20`, speedups over LRU are approximately:

- Trace 1: **1.0002×** (+0.02%)
- Trace 2: **1.0388×** (+3.88%)
- Trace 3: **1.0014×** (+0.14%)

### Interpretation

- The dynamic policy improves Trace 2 IPC by roughly 3.9%.
- Trace 1 is essentially unchanged from LRU.
- Trace 3 is only slightly better than LRU for the best dynamic configuration in this sweep.
- Thresholds 20 and 25 tie on Trace 2 at the shown precision, while threshold 20 is better on Trace 3.
- Threshold 40 hurts Trace 2 relative to the lower thresholds.
- In the current results, the dynamic policy does **not** beat `irrip_mild` on overall IPC across these three traces. It remains the Task 2B candidate because Task 2B evaluates the dynamic set-dueling design, not because it is the best policy for every trace.

### Weight-3 experiments

A weight-3 sweep was also run. The best reported IPC values for the listed threshold variants were:

| Configuration | Trace 1 IPC | Trace 2 IPC | Trace 3 IPC |
|---|---:|---:|---:|
| `dyn_irrip_weighted_t10` | 0.232327 | 0.259105 | 0.381597 |
| `dyn_irrip_weighted_t20` | 0.232327 | 0.259092 | 0.381629 |
| `dyn_irrip_weighted_t25` | 0.232327 | 0.259092 | 0.381598 |
| `dyn_irrip_weighted_t30` | 0.232327 | 0.259092 | 0.381594 |
| `dyn_irrip_weighted_t40` | 0.232327 | 0.256848 | 0.381661 |

These results support using the weight-4, threshold-20 configuration as the leading dynamic candidate among the configurations tested so far.

## 4. Additional Task 2B experiment — score-only selection

This is an **additional experiment within Task 2B**, not a separate assignment task.

The experiment disables the pressure gate in the weighted dynamic policy to test whether the gate was filtering out too many I-RRIP decisions. The pressure-gated version remains available; the score-only implementation is in:

- `replacement/dyn_irrip_weighted_w4_scoreonly.l2c_repl`

The score-only policy uses instruction-miss weight 4, decision threshold 1/4, and a 32,768-update window, with the pressure gate disabled.

### Measured results

| Trace | LRU IPC | Score-only IPC | Speedup | Total L2 MPKI | Load MPKI | Data-load MPKI | Instruction-load MPKI |
|---|---:|---:|---:|---:|---:|---:|---:|
| Trace 1 | 0.232288 | 0.232326 | 1.0002× | 23.9866 | 17.6942 | 7.1072 | 10.5870 |
| Trace 2 | 0.250638 | 0.260311 | 1.0386× | 33.9968 | 20.7497 | 11.9575 | 8.7922 |
| Trace 3 | 0.382008 | 0.382943 | 1.0024× | 8.38732 | 3.94532 | 3.2084 | 0.73692 |

Result logs:

- `results/scoreonly_Trace1-003__dyn_irrip_weighted_w4_scoreonly_w25m_s25m.log`
- `results/scoreonly_Trace2-002__dyn_irrip_weighted_w4_scoreonly_w25m_s25m.log`
- `results/scoreonly_Trace3-001__dyn_irrip_weighted_w4_scoreonly_w25m_s25m.log`

### Interpretation

- Score-only improves IPC slightly over LRU on all three traces, with the largest gain on Trace 2.
- Compared with the pressure-gated `w4_t20` policy, score-only is slightly worse on Trace 1 and Trace 2 but better on Trace 3.
- The experiment is useful as supporting analysis of the Task 2B selection mechanism. It does not establish that score-only is the best policy overall.
- Lower total MPKI does not always imply higher IPC; IPC also depends on factors such as miss type, latency, overlap, and other execution effects.

### Selection diagnostics

The score-only logs report the following leader-policy update counts and scores:

| Trace | RRIP follower update calls | I-RRIP follower update calls | RRIP score | I-RRIP score |
|---|---:|---:|---:|---:|
| Trace 1 | 1,807,306 | 1,610 | 21,102 | 20,739 |
| Trace 2 | 795,540 | 2,023,839 | 16,113 | 11,616 |
| Trace 3 | 71,837 | 939,191 | 11,906 | 11,228 |

**Note:** These are update-call counts, not counts of unique sets. They are diagnostics for understanding policy behavior, not direct measures of IPC or unique-set coverage.

## 5. Current policy choices

| Assignment part | Current candidate | Reason |
|---|---|---|
| Task 2A | `irrip_mild` | Clear instruction-aware static policy; improves IPC on Trace 2 and Trace 3, with a small loss on Trace 1 |
| Task 2B | `dyn_irrip_weighted_w4_t20` | Leading threshold-based dynamic candidate in the current IPC sweep; improves Trace 2 and is close to baseline on the other traces |
| Supporting experiment within Task 2B | `dyn_irrip_weighted_w4_scoreonly` | Tests the effect of disabling the pressure gate; useful for explaining the selection mechanism |

These are current experimental choices, not a claim that either policy wins on every trace.

## 6. Remaining work

- [ ] Extract total L2 MPKI, load MPKI, data-load MPKI, and instruction-load MPKI for the selected Task 2B threshold variants from their existing logs.
- [ ] Create the required tables and plots for Task 2A and Task 2B.
- [ ] Explain the leader-set mapping, score comparison, threshold/counter behavior, and insertion/promotion/eviction decisions in the final report.
- [ ] Verify all final reported numbers against the simulation logs.
- [ ] Assemble the required `Summary.pdf`.
- [ ] Treat Task 3 (SPP + dynamic I-RRIP) separately; it is not marked complete by this README.

## 7. Reproducibility notes

Build a policy using the repository's build script:

```bash
./build_champsim.sh no POLICY POLICY
```

Use the actual generated binary path rather than assuming the binary is named `./bin/POLICY`. The tested binary name follows the repository's ChampSim naming convention, for example:

```text
bin/hashed_perceptron-no-no-no-no-no-no-no-lru-lru-lru-POLICY-lru-lru-lru-lru-1core-POLICY
```

Run simulations manually with the configured 25M warmup and 25M measured instructions, and retain the corresponding log files. Performance claims in the report should be based on completed logs, not estimated results.

## 8. Scope clarification

The score-only experiment is an extra investigation **inside Task 2B**. It does not replace or complete a separate assignment task. Task 3 is separate and remains unvalidated here.
