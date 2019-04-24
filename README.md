# Real-Time Operating System Exercises

## Introduction

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

* [Introduction to Cyclictest and Hackbench](https://github.com/SSG-DRD-IOT/real-time-lab/blob/master/cyclictest-hackbench.md)

### Concurency
* [Introduction to PREEMPT-RT](https://github.com/SSG-DRD-IOT/real-time-lab/blob/master/concurrency/Introduction.md)
* [Affinity Tuning](https://github.com/SSG-DRD-IOT/real-time-lab/blob/master/concurrency/Affinity.md)
* [Real-Time Scheduling](https://github.com/SSG-DRD-IOT/real-time-lab/blob/master/concurrency/Scheduling.md)
* [Power Management Firmware Tuning](https://github.com/SSG-DRD-IOT/real-time-lab/blob/master/concurrency/PowerManagementFirmware.md)

### Memory and IO
* [Memory Management](https://github.com/SSG-DRD-IOT/real-time-lab/blob/master/memory-io/MemoryManagement.md)
* [Cache Allocation Technology](https://github.com/SSG-DRD-IOT/real-time-lab/blob/master/memory-io/CacheAllocationTechnology.md)

### Timing
* [High Precision Timers](https://github.com/SSG-DRD-IOT/real-time-lab/blob/master/timing/HighPrecisionTimers.md)


