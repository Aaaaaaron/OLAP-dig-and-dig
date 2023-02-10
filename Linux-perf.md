---
title: Linux 性能调优
date: 2023-02-02 19:08:04
tags:
---
# 常用命令
```
$ perf

 usage: perf [--version] [--help] [OPTIONS] COMMAND [ARGS]

 The most commonly used perf commands are:
   annotate        Read perf.data (created by perf record) and display annotated code
   archive         Create archive with object files with build-ids found in perf.data file
   bench           General framework for benchmark suites
   buildid-cache   Manage build-id cache.
   buildid-list    List the buildids in a perf.data file
   c2c             Shared Data C2C/HITM Analyzer.
   config          Get and set variables in a configuration file.
   data            Data file related processing
   diff            Read perf.data files and display the differential profile
   evlist          List the event names in a perf.data file
   ftrace          simple wrapper for kernel's ftrace functionality
   inject          Filter to augment the events stream with additional information
   kallsyms        Searches running kernel for symbols
   kmem            Tool to trace/measure kernel memory properties
   kvm             Tool to trace/measure kvm guest os
   list            List all symbolic event types
   lock            Analyze lock events
   mem             Profile memory accesses
   record          Run a command and record its profile into perf.data
   report          Read perf.data (created by perf record) and display the profile
   sched           Tool to trace/measure scheduler properties (latencies)
   script          Read perf.data (created by perf record) and display trace output
   stat            Run a command and gather performance counter statistics
   test            Runs sanity tests.
   timechart       Tool to visualize total system behavior during a workload
   top             System profiling tool.
   version         display the version of perf binary
   probe           Define new dynamic tracepoints
   trace           strace inspired tool
````

`perf list`: 产看 perf的 events

`perf top`

`perf record -a -g -p [pid]`/`perf report -g`

`perf stat -p [pid]``
开始了, 跑程序, 然后 ctrl+c
```bash
         59,665.07 msec task-clock                #   25.719 CPUs utilized
            29,686      context-switches          #    0.498 K/sec
             4,043      cpu-migrations            #    0.068 K/sec
           904,315      page-faults               #    0.015 M/sec
   182,466,072,426      cycles                    #    3.058 GHz                      (83.06%)
    48,393,560,647      stalled-cycles-frontend   #   26.52% frontend cycles idle     (83.27%)
    41,766,561,738      stalled-cycles-backend    #   22.89% backend cycles idle      (83.35%)
   127,710,645,250      instructions              #    0.70  insn per cycle
                                                  #    0.38  stalled cycles per insn  (83.58%)
    24,547,226,083      branches                  #  411.417 M/sec                    (83.73%)
       693,011,495      branch-misses             #    2.82% of all branches          (83.00%)

       2.319854477 seconds time elapsed
```

`perf stat -e L1-dcache-load-misses -e L1-dcache-load -p`

`perf stat -e L1-icache-load-misses -e L1-icache-load -p [pid]`

```bash
 Performance counter stats for process id 'xxx':

        22,036,023      L1-icache-load-misses     #    1.03% of all L1-icache hits
     2,132,515,373      L1-icache-load

       2.590468195 seconds time elapsed
```

```bash
#perf record -e cpu-clock -g -p $pid -- sleep $sec
perf record -F 99 -e cpu-clock -g -p $pid --call-graph dwarf -- sleep $sec
perf script > out.perf
/code/FlameGraph/stackcollapse-perf.pl out.perf > out.folded
/code/FlameGraph/flamegraph.pl out.folded > $file_name.svg
````

`numactl --cpubind=0` 绑定核数


