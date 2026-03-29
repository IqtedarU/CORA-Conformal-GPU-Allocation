# CORA: Cost-Aware GPU Hour Allocation via Conformal Risk Control

**Status:** Work in Progress

This repository contains the experimental notebook and evaluation code for CORA, an uncertainty-aware GPU hour allocation framework. 

*Note: The Adaptive Conformal Inference (ACI) module is currently being updated to handle temporal distribution shifts. However, the core Conformal Risk Control (CRC) and Conformalized Quantile Regression (CQR) results presented in the notebook are fully reproducible via fixed random seeds.*

## Overview

Allocating GPU time for cluster jobs is an operationally critical decision. If a job exceeds its time allocation, it is typically killed, resulting in lost compute and wasted resources. Conversely, over-allocating time using rigid, fixed multiplicative buffers results in idle reservations and low cluster efficiency. 

CORA addresses this by formulating GPU hour allocation as a decision problem under uncertainty. It combines Conformalized Quantile Regression (CQR) to generate heteroscedastic, per-job prediction intervals with Conformal Risk Control (CRC) to optimally scale these intervals subject to a formal, user-defined guarantee on expected overrun.

## Repository Contents

* `CRC_CQR_GPU_Scheduling_CORA.ipynb`: The primary Jupyter notebook containing the data processing pipeline, LightGBM quantile regression training, conformal calibration, and baseline comparisons evaluated on the Alibaba PAI trace.

## Key Methods

* **Global CORA:** Provides a single, average overrun guarantee across all jobs, minimizing total scheduling cost by assigning tighter bounds to predictable jobs and wider margins to highly uncertain ones.
* **Tail-Adaptive CORA:** GPU runtimes are heavily tail-distributed. Tail-Adaptive CORA partitions jobs into predicted-runtime buckets and applies CRC independently to each, providing stronger, per-class SLA protections.

## Current Results

The current notebook evaluates CORA against the Alibaba PAI 2020 GPU trace, comprising roughly 732,000 jobs.

### Cost Savings at Matched Coverage
CORA consistently outperforms fixed multiplicative buffers across all risk preferences (represented by cost ratio `R`, mapping the cost of a violation vs. the cost of waste). When matched to the best possible fixed buffer at equivalent coverage levels, CORA yields up to 50% savings.

| Risk Ratio (R) | Target Coverage | CORA Cost | Best Buffer Cost | Savings |
| :--- | :--- | :--- | :--- | :--- |
| **R = 2** | 82.4% | 5,823 | 6,724 | **13.4%** |
| **R = 5** | 87.5% | 10,399 | 14,021 | **25.8%** |
| **R = 10** | 92.5% | 16,056 | 26,249 | **38.8%** |
| **R = 20** | 96.2% | 24,490 | 49,027 | **50.0%** |
| **R = 50** | 98.3% | 43,432 | N/A* | **N/A** |

*\* No fixed buffer can achieve this coverage level without infinite waste.*

### Queue Dynamics and Max-Stretch
To simulate realistic cluster dynamics, allocations were evaluated in a non-clairvoyant Earliest Deadline First (EDF) queue across 50 trials of 5,000 jobs. Relying on unbuffered point predictions (EDF-P) results in widespread allocation overruns and massive queue congestion. Tail-Adaptive CORA systematically pads heavy-tailed jobs while keeping short jobs tight, achieving the lowest overall worst-case stretch.

| Allocation Method | R=2 | R=5 | R=10 | R=20 | R=50 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **EDF-P (ATLAS Baseline)** | 2461.5 | 2461.5 | 2461.5 | 2461.5 | 2461.5 |
| **Buffer 100%** | 2225.2 | 2225.2 | 2225.2 | 2225.2 | 2225.2 |
| **Global CORA** | 2611.4 | 2584.3 | 2524.7 | 2359.3 | 2183.6 |
| **Tail-Adaptive CORA** | 2397.6 | 2349.4 | 2288.7 | 2237.3 | **2145.6** |

### Tail-Adaptive Weighting Strategies
For massive jobs running longer than 24 hours, standard multiplicative buffers result in high kill rates. Tail-Adaptive CORA protects these jobs by partitioning the overrun budget across runtime classes. By weighting the target alpha by expected runtime, the framework reduces total cost while maintaining elite coverage for the heaviest tails.

*Performance of Tail-Adaptive CORA under different weighting strategies at R=50:*

| Alpha Strategy | Total Cost | Coverage | Waste | Overrun |
| :--- | :--- | :--- | :--- | :--- |
| **Equal Alpha** | 49,883 | 93.6% | 26,873 | 460 |
| **Size-weighted** | 48,455 | 90.9% | 14,188 | 685 |
| **Cost-weighted** | 47,856 | 99.1% | 30,062 | 356 |
| **Runtime-weighted** | 47,305 | 99.1% | 29,001 | 366 |
| **Global CORA (No Buckets)** | 43,432 | 98.3% | 18,412 | 500 |

## Acknowledgements

The evaluation in this repository utilizes the leakage-free Alibaba PAI GPU trace formatting provided by the ATLAS Benchmark (ICLR 2026).
