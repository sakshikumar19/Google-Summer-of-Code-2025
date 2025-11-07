<div align="center">
    <a href="https://summerofcode.withgoogle.com/"><img src="https://i.imgur.com/pgkUceb.png" width="650" alt="google-summer-of-code"></a>
    <br>
    <b> 
        <p>
        Prometheus Energy Profiler: Energy Analysis and Validation Pipeline (GSoC 2025)
        </p>
    </b>
</div>

<br>
<br>

<p align="center">
<code> <a href="#project-abstract">Project Abstract</a>&nbsp;&nbsp;&nbsp; <a href="#key-results">Key Results</a>&nbsp;&nbsp;&nbsp; <a href="#what-was-done">What Was Done</a>&nbsp;&nbsp;&nbsp; <a href="#contributions">Contributions</a>&nbsp;&nbsp;&nbsp; <a href="#discussion">Discussion</a>&nbsp;&nbsp;&nbsp; <a href="#mentors">Mentors</a>&nbsp;&nbsp;&nbsp; <a href="#links">Links</a>&nbsp;&nbsp;&nbsp; <a href="#next-steps--future-work">Next Steps / Future Work</a>
</code>
</p>

<br>
<br>

 This report summarizes my Google Summer of Code 2025 work on energy profiling and validation using Prometheus telemetry. It presents the key results, the reasoning behind them and how they inform tooling, developer workflows and future scheduling. Reproducibility and implementation details are documented in the main analysis repository.

## Project Abstract

The Prometheus Energy Profiler provides a reproducible analysis workflow to correlate power consumption with system telemetry from Prometheus monitoring systems. This project enables external validation of energy measurements and produces clear visualizations. The work includes pairwise correlation analysis across disk, memory, network and process metrics to establish baseline relationships, with a focus on power delta analysis. Beyond the analysis pipeline, contributions include revamping the MetaGreenData Django application for green computing metadata format, conducting literature review for the collaborative research paper "An environmental assessment of high-performance computing services in a school of engineering" and developing flexible node exporter scripts for real-time data collation.

## Key Results

Analysis on `wn3803100` shows several patterns that help us understand power-load relationships and workload behavior. At finer cadences, power delta vs load shows clustered, non-uniform structure:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(1min)_vs_Load_(1m)/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

This suggests instantaneous power does not track load linearly, likely due to job batching, scheduled bursts or delayed system responses.

Takeaway: short-term load is not a reliable proxy for instantaneous power.

At 5-minute aggregation, broader system behavior becomes visible:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(5min)_vs_Load_(1m)/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

The patterns suggest periodicity or phase transitions that could benefit from time-series analysis.

Takeaway: use time-series overlays or change-point detection to capture regime shifts.

The relationship between power and disk I/O shows non-linear coupling. Power Delta (5 min) vs Disk I/O Time:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(5min)_vs_Disk_IO_Time/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

This shows discontinuous clusters with gaps, indicating power can vary without corresponding disk activity, likely due to bursty or threshold-based behavior.

Takeaway: power rises can be decoupled from disk activity; look to CPU/memory.

At 1-minute resolution:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(1min)_vs_Disk_IO_Time/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

There's an inverse relationship: as power increases, disk I/O time decreases, which confirms that high-power periods are CPU-bound rather than I/O-bound. This pattern shows we can classify workload types from external telemetry.

Why it matters: classify CPU- vs I/O-bound periods without code instrumentation.

At coarser temporal scales, Power Delta (60 min) vs Load (1 m) illustrates the challenge of mismatched temporal resolution:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(60min)_vs_Load_(1m)/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

Comparing an hourly power delta to a single load snapshot requires aggregating load per hour (min/max/quartiles) to make meaningful comparisons.

Tip: summarize load per hour (min/max/quartiles) before comparing to hourly power.

Numbers at a glance (wn3803100):

- Power Delta (10 min) vs Load (1 m): Pearson 0.9968, Spearman 0.6869 (N=72)
- Power Delta (10 min) vs Disk I/O Time: Pearson -0.4065, Spearman -0.3007 (N=72)

System-wide baselines (atlaslab18):

- Disk: Reads vs Writes Pearson 0.63, Spearman 0.05 (mixed workload types)
- Load vs Disk I/O Time: weak (Pearson 0.05, Spearman 0.11) → load reflects CPU queue more than disk
- Memory: Mem Available vs Load Pearson -0.03, Spearman -0.26 (cache effects); Mem Cached vs Load ~0
- Process: Procs Running vs Load ~0; Procs Blocked vs Load ~0
- Network: RX vs TX Bytes moderate (Pearson 0.27, Spearman 0.45) after rate conversion; errors/drops ~0

Practical insights (from the above):

- If power rises while disk I/O time falls, the period is likely CPU-bound. Focus profiling on CPU paths.
- When comparing hourly power to load, aggregate load by hour first (min, max, quartiles).
- Use 5-minute windows to reveal structure, then inspect 1-minute slices to find causes.
- Convert counters to rates before correlating to avoid false positives.
- Check RX vs TX symmetry and near-zero errors to rule out network as a bottleneck.

## Discussion

### How this work fits with the wider effort

This work is part of a three-part ecosystem that enables carbon-aware computing and future job scheduling based on potential carbon cost:

- **Collation scripts (Glasgow):** Assist with data collation on nodes and provide sanity checks for data integrity.
- **Analysis scripts (this project):** Provide external monitoring and visualization to understand system behavior and possibilities.
- **Internal monitoring (mentor's ongoing work):** Embeds carbon tracking inside physics computational code, helping developers understand where their costs are and improve their code.

The collation and analysis scripts together provide external monitoring (external to the code). The internal monitoring can then be validated against this external monitoring. Internal monitoring helps developers understand where their costs are and improve, while collation scripts enable sanity checks and the analysis scripts help understand what possibilities exist for a system. The focus on one machine, rather than cluster-wide analysis, means this analysis relates to the specific instrumented code being run, which enables validation and better understanding of behavior for IO-bound software and CPU-bound software.

Where the findings help:

- CPU-bound identification: the inverse power vs disk I/O relationship lets us flag CPU-bound periods from telemetry alone, validating internal monitors and guiding code profiling.
- Scheduling signals: clustered power-load structures and periodicity hints can inform when to place jobs (e.g., align CPU-heavy work with greener windows).
- Developer feedback: per-machine correlations and plots provide a quick read on whether recent changes increased CPU pressure or shifted I/O behavior.

In combination, this enables validation (internal vs external), developer insight into cost drivers and a path toward scheduling based on potential carbon cost. The one-machine focus ties analysis to specific instrumented runs (IO-bound vs CPU-bound), which helps us understand behavior and enables validation that would be confounded in cluster-wide aggregates.

### What the results mean and questions they raise

The findings show both clear patterns and some anomalies:

- Clustered structures in power-load and power-I/O plots suggest thresholding, batching or phase changes. Which job-level features explain these clusters?
- The inverse relationship between power and I/O time supports a CPU-bound picture at high power draw. Can workload type be classified from external telemetry alone?
- Temporal scale matters. What aggregation of load (e.g., hourly min/max/quartiles) best matches hourly power deltas?

These questions motivate follow-up work with job metadata, multi-metric analysis and longer time windows to uncover the underlying mechanisms.

### Conclusions

- Delta-based analyses show power rises with CPU/memory pressure and is weakly tied to disk I/O time in these windows.
- Non-linear distributions and clustering motivate follow-ups with job metadata and multi-metric methods.
- The pipeline provides an interpretable baseline for internal-external validation and future carbon-aware scheduling experiments.
- The one-machine focus enables validation of specific instrumented runs, distinguishing IO-bound from CPU-bound behavior, which is essential for understanding energy costs at the code level.

## What Was Done

### Analysis Pipeline Development

Built a per-machine analysis workflow that:

1. **Loads multiple data formats:**

   - Prometheus matrix JSON responses (`.json` or `.json.gz`)
   - CSV power delta files (wide format with machine column)

2. **Handles metric alignment:**

   - Filters by machine instance (with normalization for port suffixes)
   - Aligns timestamps between different metrics
   - Resamples to appropriate cadence when mixing different time scales

3. **Applies statistical corrections:**

   - Automatically detects and converts cumulative/counter metrics to rates
   - Handles counter resets gracefully
   - Applies log scaling for wide dynamic ranges (improves readability)

4. **Generates reproducible outputs:**
   - Scatter plots with automatic axis scaling
   - 2D density heatmaps for data distribution visualization
   - Text summaries with Pearson/Spearman correlations, p-values and sample sizes

## Contributions

### Prometheus Energy Profiler

**Main Analysis Repository:** [prometheus-energy-profiler](https://github.com/sakshikumar19/prometheus-energy-analysis)

Core components:

- `src/load_prometheus.py` - Data loading with JSON/CSV support
- `src/sync_metrics.py` - Time alignment and rate conversion
- `src/correlations.py` - Pearson/Spearman correlation computation
- `src/visualization.py` - Scatter plots and heatmap generation
- `src/cli/run_pair_analysis` - Command-line interface

### MetaGreenData Django App Revamp

Significant contributions were made to the [MetaGreenData Django application](https://github.com/GreenAlgorithms/MetaGreenData), which manages green computing metadata. Two merged pull requests improved form validation feedback and implemented key methods and styling:

<div align="center">

|                              PR Link                               |             Description             |      Status      |
| :----------------------------------------------------------------: | :---------------------------------: | :--------------: |
| [PR #21](https://github.com/GreenAlgorithms/MetaGreenData/pull/21) |  Improved form validation feedback  | ✅ Merged  |
| [PR #20](https://github.com/GreenAlgorithms/MetaGreenData/pull/20) | Key methods and styling implemented | ✅ Merged |

</div>

These improvements help researchers and developers submitting energy and carbon metadata, making the application more robust and user-friendly for the green computing community.

### Research Contribution: Literature Review

Conducted literature review for the collaborative research paper **"An environmental assessment of high-performance computing services in a school of engineering"** with authors from the University of Manchester. The literature review covered the state of the art of environmental assessment of computing services, including indicators, metrics, hotspots and research gaps. This work bridges the gap between practical energy profiling tools and academic research on computing energy costs and is targeted for submission later this year.

### Node Exporter/Collation Scripts (Glasgow)

Developed flexible node exporter and collation scripts for Glasgow nodes that assist with data collation on nodes. These scripts provide:

- Flexible metric selection and power ingestion
- Real-time validation checks (sum of core telemetry vs IPMI/BMC measurements)
- Sanity checks for data integrity

The scripts are currently private while validation completes. Once validated, these scripts will provide infrastructure for the three-part ecosystem: collation scripts enable data collection, analysis scripts provide understanding and internal monitoring enables developer insight.

## Mentors

A huge thank you to my mentors for their guidance, patience and insights throughout GSoC.

- **Caterina Doglioni** - [GitHub](https://github.com/caterina-doglioni)
- **Michael Sparks** - [GitHub](https://github.com/sparkslabs)

Thank you also to collaborators in the "An environmental assessment of high-performance computing services in a school of engineering" research paper at the University of Manchester.

## Links

- **Main Analysis Repository:** [prometheus-energy-analysis](https://github.com/sakshikumar19/prometheus-energy-analysis)
- **MetaGreenData Contributions:**
  - [PR #21 - Improved form validation feedback](https://github.com/GreenAlgorithms/MetaGreenData/pull/21)
  - [PR #20 - Key methods and styling implemented](https://github.com/GreenAlgorithms/MetaGreenData/pull/20)

## Next Steps / Future Work

1. **Extend to multi-metric models:** Move beyond pairwise correlations to regression models that incorporate multiple system metrics simultaneously.

2. **Scheduler integration:** Add context from HTCondor/Kubernetes schedulers to relate queueing decisions and job placement to energy outcomes.

3. **Time-series analysis:** Add temporal overlays and change-point detection for power vs load relationships to inspect temporal structure.

4. **Public release of collation scripts:** After validation completes, release node exporter/collation scripts with automated cross-checks (core sum vs IPMI/BMC) and alerting.
