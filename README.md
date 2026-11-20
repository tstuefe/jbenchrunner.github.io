# jbenchrunner

A command-line harness for scheduling, monitoring and post-analyzing large groups of Java benchmarks; and to so reproducibly and in scale.

Written for JVM engineers and analysts who don't want to waste time on benchmarking logistics.

Note that even though this is pre-release V1.0, it is fully functional already.

Supported benchmarks (for now) are: 
 - [DaCapo](https://www.dacapobench.org/) 23.11 (Chopin) and
 - SPECjbb2015 (single-JVM mode).

Planned, will come soon:
  - Renaissance
  - SpecJVM
  - custom-written java apps/benchmarks

## What it does

Benchmarks are time consuming, and bad runs waste a lot of time and resources. jbenchrunner makes this a lot easier and less error-prone.

- **Define batches.** Define complex benchmark run batches in a simple, convenient markup language.
- **Schedule batches.** Run large batches of benchmark runs unattended; interrupt them if necessary and resume them later.
- **Benchmark stability.** Pin runs to CPU sets, stabilize machines for benchmark runs, provide automatic ramdisk support, schedule machine shutdowns.
- **Prevent Errors** Prevents or catches many common errors that can taint manually executed benchmarks.
- **Record everything.** Records a ton of metrics during the run: run- and machine-context, log files, and a ton of data of
  optional collectors (pidstat, perf, procfs, NMT, JVM system map, core temps, etc.)
- **Analyze.** Completed runs can be tabulated, their statistics examined, every metric compared against a base line, statistical deviations calculated etc.
- **Reporting.** Reporting allows the generation of html reports or markdown tables for further consumption.

See this `example-reports/collector-shootout-1-full/index.html` for a real-world example of a report.

## Current limitations

For now, `jbenchrunner` works out-of-the-box only with single JVM processes - any benchmark that is a single, individual process invocation. Therefore, it works well with dacapo or renaissance (the latter once we add support), but supports SpecJBB2015 only in composite mode.

Multi-process setups are possible, but require some additional work.

## Requirements

- Linux (x86_64 or aarch64). macOS works too, but with fewer
  collectors and without machine stabilization.
- Python 3.11 or newer (standard library only)
- bash
- The benchmarks themselves (see [Install](#install)), and the JDKs you want
  to test

Optional, depending on what you use. A collector whose tool is missing is
disabled by `jbr-prepare` with a warning; the other tools report it when
used:

| Tool | Package (Debian/Ubuntu) | Needed for |
|---|---|---|
| `pidstat` | `sysstat` | CPU / memory collector |
| `perf` | `linux-perf` (Debian), `linux-tools-$(uname -r)` (Ubuntu) | perf counter collector |
| `sensors` | `lm-sensors` | CPU temperature on AMD |
| `zstd` | `zstd` | compressed collector logs |
| `gnuplot` | `gnuplot` | plots |
| `jq` | `jq` | `jbr-filt --where` |
| `taskset` | `util-linux` | CPU pinning |
| `bc` | `bc` | perf collector |

## Install

### 1. Get jbenchrunner

```sh
git clone <repository-url> jbenchrunner
cd jbenchrunner
```

There is nothing to build. Add `bin/` to your `PATH`, e.g. in `~/.bashrc`:

```sh
export PATH="$PATH:/path/to/jbenchrunner/bin"
```

The commands must be called through `bin/` itself; symlinks to them (e.g.
from `~/bin`) do not work.

### 2. Install the benchmarks

- **DaCapo:** download the DaCapo 23.11 Chopin release from
  [dacapobench.org](https://www.dacapobench.org/) and unpack it, so that the
  directory contains `dacapo-23.11-chopin.jar` (or any `dacapo-*chopin.jar`).
- **SPECjbb2015:** install it from your licensed copy; the directory must
  contain `specjbb2015.jar`.

You only need the benchmarks you actually run. jbenchrunner never writes
into these directories.

### 3. Describe the machine

Machine-specific settings live in `config/hosts/<hostname>.host.toml`, where
`<hostname>` is what `hostname` prints. Without such a file,
`config/hosts/default.host.toml` is used. Start from the default:

```sh
cp config/hosts/default.host.toml config/hosts/$(hostname).host.toml
```

and set at least the benchmark paths and the largest heap the machine can
give a benchmark (in MB):

```toml
maxheap = 32768
dacapo_dir = "/opt/benchmarks/dacapo-23.11-chopin"
specjbb_dir = "/opt/benchmarks/specjbb2015"
```

The file can also set up CPU pinning and a ramdisk; see
`docs/spec-prepare.md` §1.6.

### 4. Register your JDKs

Each JDK a run can use has a file `config/jdks/<name>.jdk.toml` with its
`JAVA_HOME`. Start from the example:

```sh
cp config/jdks/example.jdk.toml config/jdks/openjdk-25.jdk.toml
```

and set the path:

```toml
java_home = "/opt/jdks/jdk-25"
```

`<name>` is what `.run.toml` files refer to (`jdkname = "openjdk-25"`);
`config/jdks/default.jdk.toml` is used when a run names none, so create that
one too. The run template `example.run.toml` uses `openjdk-25` and
`openjdk-21`. The JDK version and type (OpenJDK or OpenJ9) are detected
automatically.

### 5. Optional: allow the root commands (`--sudo`)

Only needed for machine stabilization (`jbr-prepare --sudo --stabilize-machine`)
and `jbr-execute --shutdown`. Stabilization turns off ASLR and swap, stops
services and more, until reboot: it weakens security, so it is never on by
default, and `jbr-execute` recommends a reboot afterwards.

jbenchrunner never runs as root and never asks for a password. The few
commands that need root are granted one by one, by sudoers rules for the group
`jbenchrunners`. Every process of a group member can use these rules without a
password, including the JVM under test. So keep the group small, and **run the
benchmarks as a dedicated user** that is used for nothing else:

```sh
sudo groupadd jbenchrunners
sudo useradd -m -G jbenchrunners jbench     # dedicated benchmark user (recommended)
# or, if you must: sudo usermod -aG jbenchrunners <user>

sudo visudo -cf config/jbenchrunner.sudoers
sudo install -o root -g root -m 0440 config/jbenchrunner.sudoers \
    /etc/sudoers.d/jbenchrunner
```

Group membership takes effect at the next login. The benchmark user needs read
access to jbenchrunner, the JDKs and the benchmarks, and write access to its
projects; install those where it can reach them, or set up the user's own
`PATH` and host config (steps 1 to 4) for it.

The header of `config/jbenchrunner.sudoers` has the details (e.g. granting the
rules to a single user, or leaving out the shutdown rule), and
`docs/where-we-need-sudo-and-why.md` lists every root command and why it is
needed.

### 6. Check the installation

```sh
./run-tests.sh --parallel
```

Tests that need a tool or machine feature that is not available are skipped.
The integration tests run real benchmarks with the JDK given as `test_jdk` in
the host config (or `JBR_TEST_JAVA_HOME`), and are skipped without one.

## Getting started

A *project* is a directory that holds results that belong together: all runs
you want to compare with each other. It can live anywhere.

```sh
# 1. Create a project; this also copies example.run.toml into it
jbr-init -m "Impact of Compact object headers" ~/bench/coh
cd ~/bench/coh

# 2. Describe the batch: edit example.run.toml (benchmarks, JDKs, profiles,
#    heap sizes, tags, collectors). Its comments explain every setting.

# 3. Prepare: resolves and checks everything, writes BATCH_IN_PROGRESS.json
jbr-prepare example.run.toml

# 4. Execute. Ctrl-C stops the batch (the current run is killed); running
#    jbr-execute again resumes, redoing the stopped run
jbr-execute

# In another terminal: progress of the running batch
jbr-status

# 5. Post-process: parse the logs of every finished run into result.json
jbr-pp

# 6. Look at the results

jbr-ls                                              # all runs of the project, default view
jbr-ls -o name,gc,tags,RESULTS,CPU,MEM,minflt       # ... with a custom set of result columns

 # ... and sorted and grouped and comparing average rss with baseline
jbr-ls -o name,heapsz,gc,tags,RESULTS,CPU,MEM,minflt -s name,heapsz,gc,tags -g name,heapsz --compare rssavg

jbr-filt -N dacapo.tomcat -t baseline | jbr-ls       # only some runs
jbr-ls --report first-look                          # report with plots in reports/first-look/
```

Note, `--help` is your friend. `jbr-ls` has many options

Every command has `--help`. Other commands: `jbr-failures` lists failed or
stopped runs, `jbr-retag` changes run tags, `jbr-copy` / `jbr-move` /
`jbr-move-batch` move results around.

## License

GPL-2.0-or-later; see [LICENSE.txt](LICENSE.txt).
