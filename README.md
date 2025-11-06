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
<code> <a href="#-project-abstract">Project Abstract</a>&nbsp;&nbsp;&nbsp; <a href="#-key-results">Key Results</a>&nbsp;&nbsp;&nbsp; <a href="#-what-was-done">What Was Done</a>&nbsp;&nbsp;&nbsp; <a href="#-contributions">Contributions</a>&nbsp;&nbsp;&nbsp; <a href="#-discussion">Discussion</a>&nbsp;&nbsp;&nbsp; <a href="#-mentors">Mentors</a>&nbsp;&nbsp;&nbsp; <a href="#-links">Links</a>
</code>
</p>

<br>
<br>

I worked on energy profiling and analysis for scientific computing, building tools to validate power measurements and understand system behavior through Prometheus metrics. The work involved pairwise correlation analysis across disk, memory, network and process metrics, with a focus on power delta analysis. I identified spurious correlations in cumulative metrics and confirmed real relationships between power and system load during my Google Summer of Code term.

This repository serves as a final report summary of my GSoC work and demonstrates the analysis pipeline developed for energy-aware computing.

## Project Abstract

The Prometheus Energy Profiler provides a reproducible analysis workflow to correlate power consumption with system telemetry from Prometheus monitoring systems. This project enables external validation of energy measurements and produces clear visualizations. The work includes pairwise correlation analysis across disk, memory, network and process metrics to establish baseline relationships, with a focus on power delta analysis. Beyond the analysis pipeline, contributions include revamping the MetaGreenData Django application for green computing metadata format, conducting literature review for the collaborative research paper "An environmental assessment of high-performance computing services in a school of engineering" and developing flexible node exporter scripts for real-time data collation.

## Key Results

Analysis on `wn3803100` shows several patterns that help us understand power-load relationships and workload behavior. At finer cadences, power delta vs load shows clustered, non-uniform structure:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(1min)_vs_Load_(1m)/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

This suggests instantaneous power does not track load linearly, likely due to job batching, scheduled bursts or delayed system responses. At 5-minute aggregation, broader system behavior becomes visible:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(5min)_vs_Load_(1m)/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

The patterns suggest periodicity or phase transitions that could benefit from time-series analysis.

The relationship between power and disk I/O shows non-linear coupling. Power Delta (5 min) vs Disk I/O Time:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(5min)_vs_Disk_IO_Time/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

This shows discontinuous clusters with gaps, indicating power can vary without corresponding disk activity, likely due to bursty or threshold-based behavior. At 1-minute resolution:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(1min)_vs_Disk_IO_Time/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

There's an inverse relationship: as power increases, disk I/O time decreases, which confirms that high-power periods are CPU-bound rather than I/O-bound. This pattern shows we can classify workload types from external telemetry. At coarser temporal scales, Power Delta (60 min) vs Load (1 m) illustrates the challenge of mismatched temporal resolution:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(60min)_vs_Load_(1m)/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

Comparing an hourly power delta to a single load snapshot requires aggregating load per hour (min/max/quartiles) to make meaningful comparisons.

Quantitatively, Power Delta (10 min) vs Load (1 m) on `wn3803100` shows Pearson 0.9968 and Spearman 0.6869 (N = 72), confirming strong positive correlation. Power Delta (10 min) vs Disk I/O Time shows Pearson -0.4065 and Spearman -0.3007 (N = 72), consistent with the inverse tendency observed visually.

Pairwise comparisons on `atlaslab18` establish baseline relationships across system dimensions. Disk Read vs Written Bytes shows Pearson 0.63 but Spearman 0.05 (N = 1441), indicating linear correlation exists but non-linear patterns weaken rank correlation, which suggests mixed workload types. Disk I/O Time vs Load shows weak correlation (Pearson 0.05, Spearman 0.11, N = 1441), confirming load is driven more by CPU queue depth than disk saturation, consistent with the power-I/O findings. Memory behavior shows Mem Available vs Load with Pearson -0.03 but Spearman -0.26 (N = 1441), indicating memory drops slightly as load rises but not linearly, consistent with caching behavior. Mem Cached vs Load shows near-zero correlation (Pearson 0.03, Spearman 0.04, N = 1441), suggesting cached memory is independent of load.

Process metrics show that Procs Running vs Load has near-zero correlation (Pearson -0.03, Spearman -0.03, N = 1441), confirming load reflects CPU queue depth rather than simple process enumeration. Procs Blocked vs Load also shows no meaningful relationship (Pearson 0.02, Spearman -0.03, N = 1441). Network patterns show Net RX vs TX Bytes with moderate symmetry (Pearson 0.27, Spearman 0.45, N = 1441) after rate conversion, which validates the pipeline's handling of counter metrics. Net RX Errors vs Drops shows NaN (both zero), indicating healthy network operation in the sampled window.

## Discussion

### How this work fits with the wider effort

This work is part of a three-part ecosystem that enables carbon-aware computing and future job scheduling based on potential carbon cost:

- **Collation scripts (Glasgow):** Assist with data collation on nodes and provide sanity checks for data integrity.
- **Analysis scripts (this project):** Provide external monitoring and visualization to understand system behavior and possibilities.
- **Internal monitoring (ongoing mentor work):** Embeds carbon tracking inside physics computational code, helping developers understand where their costs are and improve their code.

The collation and analysis scripts together provide external monitoring (external to the code). The internal monitoring can then be validated against this external monitoring. Internal monitoring helps developers understand where their costs are and improve, while collation scripts enable sanity checks and the analysis scripts help understand what possibilities exist for a system. The focus on one machine, rather than cluster-wide analysis, means this analysis relates to the specific instrumented code being run, which enables validation and better understanding of behavior for IO-bound software and CPU-bound software.

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
| [PR #21](https://github.com/GreenAlgorithms/MetaGreenData/pull/21) |  Improved form validation feedback  | ✅ Merged Jun 3  |
| [PR #20](https://github.com/GreenAlgorithms/MetaGreenData/pull/20) | Key methods and styling implemented | ✅ Merged May 29 |

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