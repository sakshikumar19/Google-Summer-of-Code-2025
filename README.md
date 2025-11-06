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
<code> <a href="#-project-abstract">Project Abstract</a>&nbsp;&nbsp;&nbsp; <a href="#-key-results">Key Results</a>&nbsp;&nbsp;&nbsp; <a href="#-discussion">Discussion</a>
</code>
</p>

<br>
<br>

I worked on energy profiling and analysis for scientific computing during my Google Summer of Code term. This project focuses on building tools to validate power measurements and understand system behavior through Prometheus metrics. The work involved pairwise correlation analysis across disk, memory, network and process metrics, with a focus on power delta analysis.

This repository serves as a final report summary of my GSoC work.

## Project Abstract

The Prometheus Energy Profiler provides a workflow to correlate power consumption with system telemetry from Prometheus monitoring systems. The project enables external validation of energy measurements and produces visualizations. The work includes pairwise correlation analysis across disk, memory, network and process metrics, with a focus on power delta analysis.

## Key Results

Analysis on `wn3803100` shows several patterns in power-load relationships and workload behavior. At finer cadences, power delta vs load shows clustered, non-uniform structure:

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

There's an inverse relationship: as power increases, disk I/O time decreases, which confirms that high-power periods are CPU-bound rather than I/O-bound. This pattern shows we can classify workload types from external telemetry.

Quantitatively, Power Delta (10 min) vs Load (1 m) on `wn3803100` shows Pearson 0.9968 and Spearman 0.6869 (N = 72), confirming strong positive correlation. Power Delta (10 min) vs Disk I/O Time shows Pearson -0.4065 and Spearman -0.3007 (N = 72), consistent with the inverse relationship observed visually.

Pairwise comparisons on `atlaslab18` establish baseline relationships across system dimensions. Disk Read vs Written Bytes shows Pearson 0.63 but Spearman 0.05 (N = 1441), indicating linear correlation exists but non-linear patterns weaken rank correlation, which suggests mixed workload types. Disk I/O Time vs Load shows weak correlation (Pearson 0.05, Spearman 0.11, N = 1441), confirming load is driven more by CPU queue depth than disk saturation.

## Discussion

### How this work fits with the wider effort

This work is part of a three-part ecosystem that enables carbon-aware computing and future job scheduling based on potential carbon cost:

- **Collation scripts (Glasgow):** Assist with data collation on nodes and provide sanity checks for data integrity.
- **Analysis scripts (this project):** Provide external monitoring and visualization to understand system behavior.
- **Internal monitoring (ongoing mentor work):** Embeds carbon tracking inside physics computational code, helping developers understand where their costs are.

The collation and analysis scripts together provide external monitoring. The internal monitoring can then be validated against this external monitoring. The focus on one machine, rather than cluster-wide analysis, means this analysis relates to the specific instrumented code being run, which enables validation and better understanding of behavior for IO-bound software and CPU-bound software.

### What the results mean

The findings show both clear patterns and some anomalies:

- Clustered structures in power-load and power-I/O plots suggest thresholding, batching or phase changes. Which job-level features explain these clusters?
- The inverse relationship between power and I/O time supports a CPU-bound picture at high power draw. Can workload type be classified from external telemetry alone?
- Temporal scale matters. What aggregation of load best matches hourly power deltas?

These questions motivate follow-up work with job metadata and multi-metric analysis to uncover the underlying mechanisms.

