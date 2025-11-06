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

I worked on energy profiling and analysis for scientific computing during my Google Summer of Code term. This project focuses on building tools to validate power measurements and understand system behavior through Prometheus metrics. The work involved pairwise correlation analysis across disk, memory, network and process metrics, with a focus on power delta analysis.

This repository serves as a final report summary of my GSoC work.

## Project Abstract

The Prometheus Energy Profiler provides a workflow to correlate power consumption with system telemetry from Prometheus monitoring systems. The project enables external validation of energy measurements and produces visualizations. The work includes pairwise correlation analysis across disk, memory, network and process metrics, with a focus on power delta analysis.

## Key Results

Analysis on `wn3803100` shows several patterns in power-load relationships. At finer cadences, power delta vs load shows clustered structure:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(1min)_vs_Load_(1m)/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

This suggests instantaneous power does not track load linearly, likely due to job batching or delayed system responses. At 5-minute aggregation, broader system behavior becomes visible:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(5min)_vs_Load_(1m)/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

The relationship between power and disk I/O shows non-linear coupling. Power Delta (5 min) vs Disk I/O Time:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(5min)_vs_Disk_IO_Time/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

This shows discontinuous clusters with gaps, indicating power can vary without corresponding disk activity. At 1-minute resolution:

<p align="center">
  <img src="https://raw.githubusercontent.com/sakshikumar19/prometheus-energy-analysis/main/results/wn3803100/Power_Delta_(1min)_vs_Disk_IO_Time/2025-07-31T23-00_to_2025-08-01T11-00/scatter.png" width="450" />
</p>

There's an inverse relationship: as power increases, disk I/O time decreases, which confirms that high-power periods are CPU-bound rather than I/O-bound.

Quantitatively, Power Delta (10 min) vs Load (1 m) on `wn3803100` shows Pearson 0.9968 and Spearman 0.6869 (N = 72), showing strong positive correlation. Power Delta (10 min) vs Disk I/O Time shows Pearson -0.4065 and Spearman -0.3007 (N = 72), consistent with the inverse relationship observed visually.

Pairwise comparisons on `atlaslab18` establish baseline relationships across system dimensions. Disk Read vs Written Bytes shows Pearson 0.63 but Spearman 0.05 (N = 1441), indicating linear correlation exists but non-linear patterns weaken rank correlation, which suggests mixed workload types. Disk I/O Time vs Load shows weak correlation (Pearson 0.05, Spearman 0.11, N = 1441), confirming load is driven more by CPU queue depth than disk saturation.
