# Multi‑Objective Robust Optimization of a Spark Data Pipeline - In Progress

## 1. Introduction

This project demonstrates a systematic, statistically grounded approach to optimizing a typical data engineering pipeline—an Apache Spark join of two large datasets—under multiple conflicting objectives and uncontrollable noise factors. The methodology integrates classical Design of Experiments (DoE), Taguchi robust design, and modern Bayesian optimization to find configurations that minimize execution time, memory usage, and cloud cost while maintaining stability across varying workloads and environments.

The work is intended as a portfolio piece showcasing expertise in experimental design, multi‑objective optimization, and practical data engineering.

---

## 2. Problem Definition

**Business Goal:**  
Deploy a Spark job that performs a join between two tables reliably and cost‑effectively in a cloud environment, subject to a memory constraint (≤16 GB per executor).

**Objectives (to be minimized):**
1. **Execution time** (seconds) – directly impacts user experience and resource usage.
2. **Peak memory usage** (GB) – must stay below 16 GB to avoid spills or failures.
3. **Cloud cost** ($) – a function of execution time and instance type (on‑demand vs. spot).
4. **Instability** – measured as the 95th percentile (P95) of execution time over multiple runs under varying noise, reflecting tail performance.

**Constraints:**  
- Peak memory ≤ 16 GB (hard constraint).  
- All solutions must be feasible (no crashes, no excessive spills).

**Noise Factors (uncontrollable in production):**
- Data size (small: 10 GB, medium: 100 GB, large: 1 TB)
- Data skew (uniform vs. highly skewed – Zipf distribution)
- Concurrent cluster load (idle vs. moderate background jobs)
- Instance type (on‑demand with stable performance vs. spot with potential interruptions)
- Spark version (e.g., 3.3, 3.4)

---

## 3. Factors Under Study

| Factor                 | Type         | Levels / Range                          |
|------------------------|--------------|-----------------------------------------|
| Join Type              | Categorical  | broadcast, sort‑merge, hash             |
| Number of Partitions   | Discrete     | 100, 200, 400, 800                      |
| File Format            | Categorical  | Parquet, ORC, Avro                       |
| Compression            | Ordinal      | none, snappy, gzip                       |
| Executor Memory        | Continuous   | 2 – 8 GB                                 |
| Partitioning Strategy  | Categorical  | key, hash, range                         |

---

## 4. Response Variables and Robustness Metrics

For each combination of control factors we record, under each noise condition:
- **Execution time** (s)
- **Peak memory** (GB)
- **Cloud cost** (computed as `time × instance_hourly_rate`)

From the replicates across noise conditions we derive:
- **Mean time** – average performance.
- **P95 time** – 95th percentile (non‑parametric, directly related to SLAs).
- **Standard deviation of time** – measure of variability.
- **Mean memory** – to check constraint violation.
- **Signal‑to‑Noise Ratio (SNR)** for “smaller‑is‑better”:
$$
\text{SNR} = -10 \log_{10}\left(\frac{1}{n}\sum_{i=1}^{n} y_i^2\right)
$$
  where \(y_i\) are the time measurements across noise runs. This metric penalises both high mean and high variance.

---

## 5. Experimental Strategy – Three Phases

### Phase 1: Screening (Identify Important Factors)

**Design:**  
- Control factors: an L16 orthogonal array (Taguchi) with each factor at 2 or 3 levels (some levels collapsed to fit).
- Noise factors: an L4 orthogonal array sampling four representative noise combinations.
- Total runs: 16 (control) × 4 (noise) = 64 runs.

**Analysis:**  
- For each control combination, compute mean time, P95 time, and mean memory across the four noise runs.
- Perform ANOVA on these responses to identify factors with statistically significant main effects.
- Generate main effects plots to visualise influence.
- Select the top 3‑4 factors for the next phase (likely join type, number of partitions, file format, executor memory).

**Deliverable:**  
List of influential factors and their relative importance.

---

### Phase 2: Robust Optimization (Taguchi with Outer Array)

**Objective:** Find settings that minimise P95 time and memory while satisfying the memory constraint, and are robust to noise.

**Selected factors:**  
- Join type (categorical)  
- File format (categorical)  
- Number of partitions (continuous, 100–800)  
- Executor memory (continuous, 2–8 GB)

**Design:**  
- Full factorial on categorical factors (3×3 = 9 combinations)  
- Central composite design (CCD) on continuous factors (2 factors → ~13 points)  
- Total control combinations: ~30  
- Outer array: a 2×2 full factorial on the two most impactful noise factors (e.g., data size and skew) → 4 noise conditions.  
- Total runs: ~30 × 4 = 120.

**Analysis:**  
- Fit response surface models (including quadratic terms and interactions) for:
  - P95 time
  - Mean memory
- Check memory constraint: for any predicted point with mean memory > 16 GB, discard or heavily penalise.
- Compute desirability function:
  - \(d_1\) for P95 time (target = minimum, weight high)
  - \(d_2\) for mean memory (target = minimum, weight medium, with steep penalty if >16 GB)
  - Overall desirability \(D = (d_1 \times d_2)^{1/2}\)
- Also compute SNR for each control combination using the “smaller‑is‑better” formula; compare the configuration that maximises SNR with the one that maximises desirability.

**Deliverable:**  
- A set of candidate robust configurations.
- Response surface contour plots.
- Recommendation for a “sweet spot” balancing performance and robustness.

---

### Phase 3: Sequential Tuning (Bayesian Optimisation)

**Objective:** Fine‑tune continuous parameters around the robust optimum, now considering all noise factors simultaneously without a fixed outer array.

**Method:**  
- Use **Optuna** with a multi‑objective sampler (e.g., MOTPE) to simultaneously minimise:
  - P95 time (estimated from 5 replicates under randomly sampled noise conditions)
  - Mean memory
- **Constraint:** peak memory ≤ 16 GB (enforced by rejecting trials that exceed it).
- **Search space:** narrow ranges around the Phase‑2 optimum (e.g., partitions ±200, memory ±2 GB).
- Run 50‑100 trials; each trial runs the pipeline 5 times under noise randomly drawn from the factor distributions.
- Record the **Pareto front** of non‑dominated solutions (trade‑off between time and memory).
- Present the final recommended configuration as a point on the Pareto front that best meets business priorities (e.g., the knee point).

**Deliverable:**  
- Pareto front plot.
- Final configuration with expected performance and robustness.

---

## 6. Implementation Notes (Python)

- **Design generation:** `pyDOE` for factorial and CCD arrays; custom Taguchi arrays can be hard‑coded or generated via `DoE` library.
- **Simulation / Execution:** For portfolio purposes, responses can be simulated using known Spark performance models (e.g., time ∝ data size / partitions + constant). Alternatively, use a real Spark cluster (e.g., AWS EMR) and measure actual runs.
- **Analysis:** `statsmodels` for ANOVA and regression; `scikit‑learn` for response surface models; `optuna` for Bayesian optimisation.
- **Visualisation:** `matplotlib`, `seaborn` for main effects, contour plots, Pareto fronts.

---

## 7. Results and Deliverables (Expected)

- **Phase 1:** Main effects plot showing, for example, that join type dominates time, while number of partitions strongly affects memory.
- **Phase 2:** A response surface highlighting a region where P95 time is low and memory is within limits; the SNR analysis confirms that the same region is robust.
- **Phase 3:** Pareto front illustrating the trade‑off between time and memory; final configuration (e.g., broadcast join, Parquet, 400 partitions, 6 GB memory) achieving P95 time of 120 s and memory 12 GB.

All code, data, and notebooks will be organised in a GitHub repository with a clear README.

---

## 8. Conclusion

This project demonstrates a rigorous, multi‑stage optimisation framework that bridges classical DoE and modern machine learning‑based tuning. It is directly applicable to real‑world data engineering tasks where performance, cost, and reliability must be balanced. The methodology is general and can be adapted to any system with tunable parameters and uncontrollable noise.
