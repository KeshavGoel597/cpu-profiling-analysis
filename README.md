# 🔍 CPU Performance Profiling & Analysis

Comprehensive performance profiling study using **gprof**, **perf**, **Callgrind/KCachegrind**, and **Flame Graphs** to analyze and identify bottlenecks in two real-world applications: a log analyzer and an image blur filter.

## Overview

This project demonstrates proficiency with the Linux performance profiling toolchain by analyzing two deliberately unoptimized programs:

- **Part A — Log Analyzer**: String-heavy parsing application bottlenecked by `parse_line` (~50% of execution), revealing hidden costs of `malloc`/`free` per line, SSE2 string searches, and redundant `strcmp` loops
- **Part B — Image Blur Filter**: Cache-unfriendly convolution kernel with **59% cache miss rate** due to column-major traversal of row-major data, plus a hidden hotspot (`init_gaussian_kernel` called per-pixel instead of once)

## Profiling Tools Used

| Tool | Type | Key Strength |
|------|------|-------------|
| **gprof** | Hybrid (sampling + instrumentation) | Call counts and basic call graph |
| **perf** | Sampling (hardware counters) | Low overhead, captures kernel/library time |
| **Callgrind** | Full emulation (Valgrind) | Exact instruction counts, finds redundant calls |
| **Flame Graphs** | Visualization | Intuitive hierarchical view of call stacks |

## Key Findings

### Part A: Log Analyzer
- `parse_line` dominates at ~50% — internally calls `malloc`/`free` per line, `strstr`, `strncpy`
- `search_messages` hidden cost: `__strstr_sse2_unaligned` at 17.6% (only visible via `perf`)
- Callgrind revealed structural inefficiency: per-line heap allocation
- **Amdahl's Law**: 10× optimization of hotspot yields only **1.82× overall speedup**

### Part B: Image Filter
- **59% cache miss rate** in `apply_blur_slow` — column-major access on row-major data
- Hidden hotspot: `init_gaussian_kernel` recomputed **millions of times** (once per pixel)
- Scaling: 256×256 → 4096×4096 shows near-linear growth, limited by memory bandwidth
- **Amdahl's Law**: 5× optimization yields **2.78× speedup** (80% parallelizable fraction)

### ROI Analysis
Part B offers **better ROI** for optimization effort — higher parallelizable fraction (80% vs 50%) and algorithmically simpler fixes.

## Project Structure

```
├── part-a/
│   ├── gprof_report.txt           # gprof flat profile and call graph
│   ├── perf_report.txt            # perf stat/record output
│   ├── flamegraph.svg             # Interactive flame graph visualization
│   └── kcachegrind_screenshot.png # Callgrind analysis screenshot
├── part-b/
│   ├── perf_stat_output.txt       # Cache miss statistics
│   ├── benchmark_results.txt      # Scaling benchmarks (256→4096)
│   └── callgraph_screenshot.png   # Call graph showing hidden hotspot
├── answers.txt                     # Complete analysis answers
└── analysis_report.pdf            # Compiled analysis report
```

## Tool Comparison Summary

| Aspect | perf (Sampling) | gprof (Hybrid) | Callgrind (Instrumentation) |
|--------|-----------------|----------------|---------------------------|
| Overhead | ~1–5% | ~10–30% | 50–100× slowdown |
| Library profiling | ✅ Yes | ❌ No | ✅ Yes |
| Exact call counts | ❌ No | ✅ Yes | ✅ Yes |
| Short function detection | ❌ May miss | ❌ Shows 0.00s | ✅ Accurate |
| Production use | ✅ Safe | ⚠️ Needs -pg | ❌ Too slow |

## Amdahl's Law Applications

```
Speedup = 1 / ((1 - P) + P/S)

Part A: P = 0.50, S = 10  →  Speedup = 1.82×
Part B: P = 0.80, S = 5   →  Speedup = 2.78×
4-core parallel (Part A):  →  Speedup = 1.60×
4-core parallel (Part B):  →  Speedup = 2.50×
```

## License

Academic project — IIIT Hyderabad, Software Systems for Programmers (SPP), Spring 2026.
