# Power Management Tuning Linux

> NOTE: Power Managenment Tuning is kind of hard to test, as of e.g. changing the C-States dynamically will have no direct influence on short tests e.g. with cyclictest, I think. 

> Usually the need of enabling/disabling of certain C-states is very application dependent. You always need to make a tradeoff between realtime<->energy consumption. Consider the following thought:

> Your rt-application needs to execute at a microsecond accuracy for start time and deadline, but within a very large scheduled interval (e.g. every week or several hours). Here it would not make sense to e.g. disable the idle C-States due to a much higer energy consumption between two executions. 

> In such situations it would be better to dynamically e.g. set the  frequency of the CPU manually to max performance, when the rt-app needs to be executed.   

The aim of Power Management is optimize energy utilization and at the same time keeping the performance of a system at a level that matches the current requirements. Thus, Power management is key for balancing the performance needs and power saving options for a system.

The power management options helps save energy but may introduce latency which is critical for real-time applications. Therefore, the power management options should be selected cautiously and a good analysis of latency impact on the system helps to optimally use these options.


## Processor Idle States(C-State)

When there is nothing to do with the CPU, it will enter into idle state, There are several CPU idle states generally called as C-states. The C-state values ranges from C0 to C6. The C0 state is the normal operating state and any increase in the C number the CPU goes into more deep sleep states and saves more power.

We can use C-state attributes to select the C-state and it again depends on our application. The C-states can be selected based on below C-state behaviour

* Exit latency of a C-state - Deeper C-states have higher exit latency. So performance issue will occur for small periodic tasks.
* Target residency of a C-state - The C-states with deeper depths need to be idle longer to compensate the energy spent on entering and exiting.

The transition between the idle states is controlled by the CPUIdle driver.

To put CPU in power saving mode when there is nothing to do the below Linux boot option can be enabled
```
      CONFIG_APM_CPU_IDLE=y
```

To enable Advanced power management options at boot time we have to enable the below option.
```
      CONFIG_APM_DO_ENABLE=y
```
The below command is to get the maximum number of C-states in our system
```
root@intel-corei7-64:~# cat /sys/module/intel_idle/parameters/max_cstate

0
```	

Real-Time application configurations avoid the power management completely as it involes considerable latency which increases with more  power conservation. The deeper the C-states more the power saved and at the same time causes more exit latency. 

To address this issue the CPU idle states governance can be tuned to efficiently select idle states that are below latency tolerance threshold.

The applications can write the latency tolerance values in microseconds to the file `/dev/cpu_dma_latency`, this sets the latency tolerance value throughput the system. The handle to the file should be kept open till the the particular power saving option is needed. 

To completely diable the C-states the applications can write 0 to the file.

## Processor Performance States(P-State)

P-states are the operational states of CPU frequency and voltage. P-state is both frequency and voltage operating point. The higher p-state implies low frequency and power at which processor runs. The highest performance state is P0.The number of p-states are depends on the type of the processor. Higher P-state means slower processor speeds and lower power consumption. To work in any P-states the processor must be in C0 state that is not in idle mode. The P-states behavior can be influenced with intel_pstate driver which is also called as pstate power scaling driver. We can disable this driver using the below boot option
```
	intel_pstate=disable
```

## Speedstep

Intel SpeedStep Technology is used to dynamically control the CPU clock speed as per the load demand. The CPU clock speed is throttled for workloads with less demand which will reduce the power consumption of the CPU.

In Linux, CPUfreq module provides an interface between low level CPU-specific frequency-control technologies and high-level CPU frequency-controlling policies. The CPUfreq enables changing of policy governors which can change the CPU frequency based on differenct criteria like CPU usage. Below is a list of governors for selecting the best Operating Performance Point. 

* PowerSave Governor - This governor always saves the power by selecting the lowest Frequency.
* Performance Governor - This governor always selects the highest frequencies thus provides high performance.
* Ondemand Governor - This governor changes frequency based on CPU utilization. It tracks the load and tracks the usage and scale the OPP's.

To see the list of governors available we can use below command
```
root@intel-corei7-64:~# cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors

conservative userspace powersave ondemand performance
```

To check current governor configuration for cpu0:
```
root@intel-corei7-64:~# cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

performance
```

To activate a particular governor we can use below command
```
root@intel-corei7-64:~# echo governor > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```
where governer mentions the new cpu frequency governor

\* - replace with cpu number


To check max CPU freq from command line interface
```
root@intel-corei7-64:~# cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq

1101000
```

To check min CPU freq from command line interface
```
root@intel-corei7-64:~# cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_min_freq

800000
```

To check current CPU freq from cli
```
root@intel-corei7-64:~# cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq

800000
```

## Active State Power Management(ASPM)

Active state power management helps power manage the PCI express links.

Check avlailable policies

```
root@intel-corei7-64:~# cat /sys/module/pcie_aspm/parameters/policy

[default] performance powersave
```
Disable via kernel boot parameter: pcie_aspm=off

## Real-Time Throttling

In real-time applications if there is any programming failures then it can cause the system to hang. Such a failure is similar to infinite loop. If the real-time task has the highest priority then no other task can preempt it. This causes real-time system block all other tasks and it will use 100% CPU. For this Linux has a safeguard mechanism that allows the system to allocate bandwidth for both real-time and non-real-time tasks. This mechanism is known as real-time scheduler throttling and is controlled by two parameters in the /proc file system.

```
root@intel-corei7-64:~# cat /proc/sys/kernel/sched_rt_period_us

1000000
```

The above file contains the period (in microseconds) to be consider as 100% of CPU bandwidth. The default value is 1000000 microseconds

```
root@intel-corei7-64:~# cat /proc/sys/kernel/sched_rt_runtime_us 

950000
```

The above file contains total bandwidth available to all real-time tasks(process with scheduling policy SCHED_FIFO or SCHED_RR or SCHED_DEADLINE). The default value is 950000 microseconds, or 95% of the CPU bandwidth.
We can set this value to -1 means that real-time tasks can use up to 100% of CPU time.
```
root@intel-corei7-64:~# echo -1 > /proc/sys/kernel/sched_rt_runtime_us
```

### RT_RUNTIME_GREED Feature

Using RT throttling we are giving like 95% CPU bandwidth to real-time tasks and 5% CPU time to non-real-time tasks(process with scheduling policy SCHED_OTHER or SCHED_NORMAL). An advanced user may want to allow the real-time task to continue running in the absence of non-realtime tasks starving. So if there is no non-real-time tasks then we have to use complete bandwidth for real-time tasks.

Before throttling the RT_RUNTIME_GREED feature checks for the non-real-time tasks. As soon as system goes idle it will unthrottle the real-time tasks.

Enable the RT_RUNTIME_GREED with the following command
```
root@intel-corei7-64:~# echo RT_RUNTIME_GREED > /sys/kernel/debug/sched_features
```
