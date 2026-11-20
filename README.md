# JbenchRunner

A command-line harness for scheduling, monitoring, and post-analyzing possibly large sets of Java benchmarks.

To do so reproducibly, scientifically sound, automated and in scale.

Written for JVM engineers and performance analysts who don't want to waste time on benchmarking logistics.

## What it does

It provides a harness to run industry-standard benchmarks (for now, DaCapo and SpecJVM, Renaissance and others will soon follow).

It allows you to:

- **Define batches.** Define complex benchmark sets in a simple, convenient markup language.
- **Schedule batches.** Run large batches of benchmark runs unattended, interrupt and later resume batch runs.
- **Benchmark stability.** Provides facilities to stabilize benchmarks and to make them reproducible: pinning JVMs to CPU sets, stabilize machines between individual runs, provide automated ramdisk support, schedule machine shutdowns ...
- **Prevent Errors** Prevents or catches many common errors that could otherwise quietly taint your manually executed benchmarks.
- **Record everything.** Records a ton of metrics during the run: run- and machine-context, log files, and a ton of data of
  optional collectors (pidstat, perf, procfs, NMT, JVM system map, core temps, etc.)
- **Analyze.** Completed run sets can be tabulated, their statistics examined, every metric compared against a base line, statistical deviations calculated etc.
- **Reporting.** Reporting allows the generation of html reports or markdown tables for further consumption.

For an example of a finished real-world batch, and a demonstration of the reporting, see [this example report](example-reports/collector-shootout-1-full/index.html)

Installation, Getting Started, documentation: Please see the [Github Project Page](https://github.com/tstuefe/jbenchrunner).

