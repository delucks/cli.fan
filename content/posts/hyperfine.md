---
title: "hyperfine"
date: 2019-03-21T14:50:23-04:00
description: "hyperfine: a shell benchmarking utility"
draft: true
---

# a benchmarking utility

`hyperfine` is a shell benchmarking tool that provides statistics of a command's execution time. It iterates on the work of previous tools like [bench](https://github.com/Gabriel439/bench) by allowing for pre-run and pre-benchmark commands to reset the state of the system. It's cross-platform and provides both human and machine-readable output.

I'm covering `hyperfine` early in [cli.fan](https://cli.fan) so subsequent posts can use it to compare similar tools.

![hyperfine overview](/hyperfine_main.png)

| Quick Facts | |
| ---- | ----------- |
| Version Control | git on [github](https://github.com/sharkdp/hyperfine/) |
| Author | David Peter ([sharkdp](https://david-peter.de/)) |
| Language | Rust |
| Binary Size | `3.5M` |

## Installation

`hyperfine` is distributed as a [statically built binary](https://github.com/sharkdp/hyperfine/releases) for x86 Windows, Linux, and Mac OSX systems. Yes, it runs on Windows! No, I haven't tried it.

It's also available in the following package managers:

| Platform | Installation Command |
| -------- | -------------------- |
| HomeBrew | `brew install hyperfine` |
| Cargo | `cargo install hyperfine` |
| AUR | `yaort -S hyperfine` |
| Void Linux | `xbps-install -S hyperfine` |

## Usage

The simplest usage of `hyperfine` is timing a single command. Here's the output of `hyperfine ls` on the source of this blog.

```text
$ hyperfine ls
Benchmark #1: ls
  Time (mean ± σ):       0.7 ms ±   0.1 ms    [User: 0.6 ms, System: 0.1 ms]
  Range (min … max):     0.4 ms …   2.7 ms    2125 runs

  Warning: Command took less than 5 ms to complete. Results might be inaccurate.
  Warning: Statistical outliers were detected. Consider re-running this benchmark on a quiet PC without any interferences from other programs. It might help to use the '--warmup' or '--prepare' options.
```

A couple things to note there- first, you're only seeing the standard output of this command. A progressbar is displayed while measuring the results:

![Animated hyperfine progressbar image](/hyperfine_progressbar.svg)

Second, the two warnings at the bottom tell you about `hyperfine`'s statistical methodology. `hyperfine` attemps calculation of the mean time of execution and the standard deviation. When a command completes very quickly (like `ls` in a directory with only 12 entries), these results may not represent the timing of the command very well. Likewise, outliers can substantially affect these non-stable statistics.

Before discussing how to reduce the influence of outliers, I want to draw your attention to the run count in the snippet above, beneath `User: 0.6ms`. For this program's timing run, `hyperfine` decided to run `ls` **2125 times**. `hyperfine` will automatically adjust the count of runs depending on the time for each individual run. By default, `hyperfine` chooses ten runs as the lower bound and will adjust upward if the command completes quickly. You can override this behavior with the `-r`/`--runs` flag, which specifies an absolute count of runs of the program to execute. The `-m` and `-M` flags let you specify a minimum and maximum count of runs to express a range.

```text
$ hyperfine -r 2 ls
Benchmark #1: ls
  Time (mean ± σ):       0.8 ms ±   0.1 ms    [User: 0.9 ms, System: 0.0 ms]
  Range (min … max):     0.7 ms …   0.9 ms    2 runs

  Warning: Command took less than 5 ms to complete. Results might be inaccurate.
```

### Benchmarking

This `ls` command is a poor example for `hyperfine` since it's not doing any computationally expensive work. Let's construct a command which opens a lot of files on our disk.

```text
$ hyperfine "find /usr/share/man -type f -name '*.gz' | xargs -L 1 -P 8 zcat"
Benchmark #1: find /usr/share/man -type f -name '*.gz' | xargs -L 1 -P 8 zcat
  Time (mean ± σ):      2.592 s ±  0.071 s    [User: 6.333 s, System: 1.786 s]
  Range (min … max):    2.523 s …  2.705 s    10 runs

```

That's more like it! This snippet introduces another feature of `hyperfine`: **support for arbitrary shell pipelines**. Every command is evaluated in its own spawned instance of `sh -c`. You can use other shells that support `-c` with the `hyperfine -S` flag.

This is cool but hey, I have a modern unix machine and there are probably disk caches getting in the way of an accurate measurement. We can run commands before each run to reset the state of the system with the `-p`/`--prepare` flag.

```text
$ hyperfine --prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches' "find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zcat"
Benchmark #1: find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zcat
  Time (mean ± σ):      3.384 s ±  0.496 s    [User: 6.807 s, System: 2.184 s]
  Range (min … max):    2.981 s …  4.364 s    10 runs

```

Looks like the disk's caches were really doing their job- the fastest run with cold caches was slower than the slowest warm iteration. There's far more standard deviation on this run though, possibly due to the state of the rest of my system. If you are looking to do "real science" with `hyperfine`, it'd be prudent to run benchmarks from a system that has very little else running.

In addition to clearing caches via commands before each benchmarking run, `hyperfine` allows you to warm caches by running throw-away warmup runs of each command. You can specify the count of warmup runs with `-w`/`--warmup`.

The shell pipeline we've been timing isn't very useful, unless you want to dump the contents of all man pages stored in `/usr/share/man`. Let's write a shitty `man -k` replacment instead:

```text
$ hyperfine --prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches' "find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zgrep syslog"
Benchmark #1: find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zgrep syslog
Error: Command terminated with non-zero exit code. Use the '-i'/'--ignore-failure' option if you want to ignore this. Alternatively, use the '--show-output' option to debug what went wrong.
```

Whoops! I forgot that if `grep` doesn't match it'll return a non-zero exit code. We could run `hyperfine` with the `--ignore-failure` option, but this could mask other possible issues with the rest of the pipeline. Since `hyperfine` supports arbitrary shell pipelines I'll just tack on a `|| true`:

```text
$ hyperfine --prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches' "find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zgrep syslog || true"
Benchmark #1: find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zgrep syslog || true
  Time (mean ± σ):      8.422 s ±  0.456 s    [User: 22.107 s, System: 3.823 s]
  Range (min … max):    7.844 s …  9.152 s    10 runs

```

Cool, we just created a _really_ naive `man -k` implementation. How fast is it comapred to the real thing? `hyperfine` can comparing multiple pipelines that are passed as additional arguments, so let's pass `man -k syslog` too:

```
$ hyperfine -p 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches' "man -k syslog" "find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zgrep syslog || true"
Benchmark #1: man -k syslog
  Time (mean ± σ):      84.6 ms ±  13.2 ms    [User: 35.4 ms, System: 12.4 ms]
  Range (min … max):    70.3 ms … 107.9 ms    14 runs

Benchmark #2: find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zgrep syslog || true
  Time (mean ± σ):      7.401 s ±  0.056 s    [User: 20.915 s, System: 3.334 s]
  Range (min … max):    7.315 s …  7.459 s    10 runs

Summary
  'man -k syslog' ran
   87.51 ± 13.64 times faster than 'find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zgrep syslog || true'

```

Since these two commands had vastly different timings, the confidence interval is a bit wide. A good thing to note here is that the `--prepare` command is run before all benchmrks, you don't have to pass multiple `--prepare` if you pass multiple commands.

### Output Formats

`hyperfine` supports outputting benchmarks in machine and human-readable formats. With `--export-markdown`, you can produce markdown tables from benchmarking runs. The following table (and the rest of the examples in this section) correspond to the last benchmark shown above.

| Command | Mean [ms] | Min…Max [ms] |
|:---|---:|---:|
| `man -k syslog` | 82.4 ± 8.9 | 70.1…99.6 |
| `find /usr/share/man/ -type f -name '*.gz' \| xargs -L 1 -P 8 zgrep syslog \|\| true` | 7177.2 ± 297.1 | 6943.1…7918.3 |

The same basic summary information is available in the CSV output, with `--export-csv`:

```text
command,mean,stddev,user,system,min,max
man -k syslog,0.07754731863714286,0.00718055888822199,0.03314085214285714,0.008886727857142855,0.06651294728,0.08886895328000001
find /usr/share/man/ -type f -name '*.gz' | xargs -L 1 -P 8 zgrep syslog || true,7.723293876680001,0.3837451768952995,21.587729494999998,3.5906936849999993,7.35156860228,8.62358789528
```

If you're looking to dig into the statistics yourself, the `--export-json` option is most useful. The JSON output format includes the timing of each individual run with the summary statistics.

```text
$ jq .results[].times hyperfine_results.json
[
  0.08253860528000001,
  0.08350332128,
  0.08733465628,
  0.08886895328000001,
  0.08629350028,
  0.07690511928,
  0.07379255728,
  0.07013369328,
  0.06651294728,
  0.07266715928,
  0.07053606928,
  0.07827986928000001,
  0.07743288128,
  0.07086312828
]
[
  8.62358789528,
  7.9401124772800005,
  7.51673595728,
  7.45856522628,
  7.48676654828,
  7.761782671280001,
  7.41736586028,
  7.69780088228,
  7.9786526462800005,
  7.35156860228
]
```

### Parameterization

One interesting feature I haven't found much of a use for is `-P`/`--parameter-scan`. This allows you to specify an integer range that will be substituted within each program timing run.

For a contrived example, let's say we wanted to find the `man` section which returns results slowest. My hypothesis is that the section with the most results will take the longest to run. A common-enough regex is probably `.*tool.*`, as many man page descriptions include the string "tool".

```text
$ for n in {1..8}; do echo -n "$n "; apropos -s "$n" '.*tool.*' 2>/dev/null | wc -l; done
1 152
2 0
3 181
4 0
5 5
6 0
7 3
8 32
```

If my hypothesis is correct, section 3 should run the slowest. Let's set up the command-line using the `{SECTION}` placeholder, like so: `hyperfine -P SECTION 1 8 --export-markdown apropos.md -i 'apropos -s {SECTION} ".*tool.*"'`

![hyperfine parameter scan animation](/hyperfine_parameter.svg)

| Command | Mean [ms] | Min…Max [ms] |
|:---|---:|---:|
| `apropos -s 1 ".*tool.*"` | 25.5 ± 3.9 | 23.7…64.3 |
| `apropos -s 2 ".*tool.*"` | 18.1 ± 0.9 | 17.3…21.9 |
| `apropos -s 3 ".*tool.*"` | 31.7 ± 1.3 | 30.5…37.6 |
| `apropos -s 4 ".*tool.*"` | 16.8 ± 0.8 | 16.0…20.6 |
| `apropos -s 5 ".*tool.*"` | 18.1 ± 1.0 | 17.2…23.9 |
| `apropos -s 6 ".*tool.*"` | 17.2 ± 2.8 | 15.9…44.9 |
| `apropos -s 7 ".*tool.*"` | 19.4 ± 3.1 | 17.3…38.8 |
| `apropos -s 8 ".*tool.*"` | 19.6 ± 0.4 | 19.0…23.0 |

The benchmarks match our hypothesis: the slowest sections are 3, 1, 8, and 7- the exact order of result count.

I'm interested in hearing your use-case for `-P` if you have one- it's clearly powerful but difficult to think about in the abstract.

The other notion of "parameterization" I've seen used with `hyperfine` is the use of shell variables in the commands you're running. This is easy to perform but won't change per-run of the command.

### Debugging Failing Commands

The other two flags I'll mention here help with debugging failing commands. You can view the standard output of the command with `--show-output`, which slows down the execution but can pinpoint exactly what input broke your command. If this occurs deep in a sequence of failures, the `-i`/`--ignore-failure` flag will stop `hyperfine` from exiting upon a non-zero exit code.

Personally, I would recommend running the command you intend on benchmarking manually before running it under `hyperfine`. If all went well beforehand and failures start cropping up during benchmarking, the `-i` and `--show-output` flags are useful to know about.

## tl;dr

`hyperfine` provides a nice balance of features against binary size and ease of installation. The JSON output format and `--prepare` feature make it worth trying for your next benchmarking use-case.

:information_source: **Interested in seeing more posts like this?** Subscribe to the [cli.fan RSS feed](/posts/index.xml) linked here and in the top navigation bar.

