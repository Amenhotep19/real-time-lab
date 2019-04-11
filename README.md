# Real-Time Operating System Exercises

## Introduction

introduce the idea of a real-time system

Unexplained delays while processing events are one of the biggest
problems in real-time operating systems. This labs will bring you through a series of exercises to benchmark and gauge different latencies within the system.

There are many types of latency within an operating system.
- the time between when an interrupt occurs and the thread
  waiting for that interrupt is run

- the time between a timer expiration and the thread waiting for
  that timer to run

- The time between the receipt of a network packet and when the
  thread waiting for that packet runs

Usually, a tool is written to test a specific latency. In this lab, we will become familiar with a couple of these tools.

## Table of Contents



## Cyclictest

***cyclictest*** is a benchmarking tool widely used to tune Linux real-time systems.

It runs a simple algorithm.
```c
while (true)
   next = now() + interval
   sleep(interval)
   actual = | now() - next |
   update_statistics(actual)
```
The statistics collected allow to identify worst case execution times, print a detailed histogram, etc.
Despite its simplicity, it allows to capture most of the factors affecting real-time performance on Linux.

Let's try running cyclictest with no command line arguments.
```console
./cyclictest
```
You should see output like this.
```console
$ cyclictest
# /dev/cpu_dma_latency set to 0us
policy: other/other: loadavg: 0.22 0.12 0.05 1/164 1276

T: 0 ( 1276) P:0 I:1000 C:  3166 Min:     19 Act:   24 Avg:   25 Max:      242
```

### Command Line Option for Cyclictest

When running without parameters, cyclictest will enter an infinite loop where it sleeps and wakes up at periodic intervals.
Looking at the last line, we can already identify some details:


* T: 0 (1276): it creates a single thread (enumerated as 0), with the identificator 1276. We can instruct cyclictest to create more threads.

* I: 1000: the time that it will sleep between measurements will be of 1000 us. This is the default value, but we can provide any period for our tests.

* C: 3166: it has been executed 3166. This figure will of course be increasing overtime. By default it will run forever, but we can set a defined number of experiments to be run. It is relevant to get enough statistical data.

* Act: 24: it is the actual value measured for the current iteration, in microseconds.

* Min: 19, Avg: 25 and Max: 242: are statistic built upon all the actual values recorded over time. That is, the minimum, average and maximum times of the difference between the time when the timer was programmed, and the time when it was actually effective, expressed in microseconds. Actually we should never take the average values into consideration, and focus on the histogram data provided by cyclictest. We will look into that in further examples.

The remaining items in the output are:

* Policy: other/other: and P: 0: they are job scheduling parameters, indicating the scheduling policy for the execution of the thread, and its priority. The values provided in this example indicate that no real-time scheduling policy has been applied. We will see how we can configure this value.

* /dev/cpu_dma_latency set to 0us: is a somehow obscure message that indicates that cyclictest has disabled certain power saving configuration options that may impact the latency. The specific behaviour can be configured, but this is the default option.

* loadavg: 0.01 0.07 0.02 1/164 1742: this is equivalent to examining the /proc/loadavg system file under Linux. 0.01 0.07 0.02 measure the CPU and IO load averaged over the last 1, 5 and 15 minutes respectively. 1/164. 1742 is the PID of the most recently process created on the system.

* If you press Ctrl + C, cyclictest will terminate.
As usual, cyclictest -h will output the list of options with a small description.
We are now done with this brief introduction to cyclictest.

### Experienting with cyclictest

Let's run cyclictest on only 1 CPU using the -a, --affinity flag:
```console
cyclictest -A 0
```
In a quad-core system the processors are labeled 0 to 3.
This runs measure thread context switching on processor 0.

To measure latencies, Cyclictest runs a non real-time master thread (scheduling class **SCHED_OTHER**) which starts a defined number of measuring threads with a defined real-time priority (scheduling class **SCHED_FIFO**). The measuring threads are woken up periodically with a defined interval by an expiring timer (cyclic alarm). Subsequently, the difference between the programmed and the effective wake-up time is calculated and handed over to the master thread via shared memory. The master thread tracks the latency values and prints the minimum, maximum, and average latencies after each iteration (default) or once the number of iterations specified is completed (–quiet).

#### Why does **ps** show Cyclictest under a normal policy (SCHED_OTHER) not a real-time policy?

The Cyclictest task always has one or more threads, several measuring threads which are scheduled under a real-time policy and a main thread (used to aggregate the measurements) which is scheduled under a normal policy. The command ps -ce only shows the main process and not its measuring threads. To also see the measuring threads, use the command ps -eLc | grep cyclic which shows the main-process and all its threads which are indicated as being scheduled under a real-time policy (SCHED_FIFO). An example of output from the correct command is below. The first column is PID (process ID), the second column is LWP (light weight process ID, thread ID), the third column is CLS (scheduling class, scheduling policy), and the fourth column is PRI (priority).

#### Observing the Thread Scheduling Algorithm

Cyclictest has many options for adjusting how measurements are made,
such as how many measurement threads are run, scheduling policies for
measurement threads, etc., which we're going to use in the next
section to try and pinpoint excessive latency sources within the
kernel.

The command launches 5 thread at a priority of 80 and sets the interval of threat to 10,000.
```console
./cyclictest -t 5 -p 80 -n -i 10000
```

We can observer the measuring thread and the real-time threads by opening another terminal and typing:

```console
ps -cLe | grep cyclic
```

Another useful comand line switch is the --smp switch which starts one thread measurer on each CPU core.

### Test Across Multi-Symmetric Processors
This flag will start a thread on each CPU core available to the system.

```console
cyclictest --smp
```
#### Set Thread Priority
You may also want to test the interrupt priority latency under different thread priority conditions. To set the initial thread priority use the -p flag. The threat priorty may be set from 0 to 99.
```console
cyclictest --smp -p95
```

#### Conditions for Ending the Latency test
There are two flags that allow you to terminate the latency tests.

The first is the -b or --breaktrace flag which stops the test if the latency of a context switch is greater than the number passed in on the command line. This can be useful is you'd like to determine what types of loads will cause the latency to go above a certain level.

```console
cyclictest --smp -p95 -b 60
```

Let's open a new terminal and use the stress command to put some extra load on the machine. This will start calculating random sqrt() functions in 128 threads
```console
stress -c 128
```

The second method of terminating the latency test is to use the -l or --loops flag. This flag causes cyclictest to run a specific number of content switches.
```console
cyclictest --smp -p95 -b 60
```


## Hackbench

## Stress
