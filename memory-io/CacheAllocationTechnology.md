# Cache Allocation Technology

Cache Allocation Technology(CAT) allows an operating system to control the allocation of a CPU’s shared last-level cache (LLC). CAT is also referred to as the platform quality of service (PQoS) and allows cache allocation on a per-core basis.

### Intel Cache Hierarchy
Intel’s Atom processors include two levels of cache: the L1, and L2 caches. The L1 cache is the smallest, but fastest, cache and is located nearest to the core. The L2 cache, also known as the last level cache (LLC), is many times larger than the L1 cache, but is not capable of the same bandwidth and low latency as the L1 cache.

### Intel Cache Policies
Each physical core contains its own private L1 caches. However, the LLC is shared and can be fully accessed and utilized by all cores in the system. If a request for data misses a core's L1, the request then continues to the LLC to be serviced. In the case of an LLC miss, the request is serviced in memory(RAM).

In almost every case, as cores request data from memory a copy is placed in each of the caches as well. As a result, when a request misses the L1, it may hit in the LLC. However, as an LLC cache line is evicted the cache line must also be invalidated in the L1 if they exist. This can happen, for example, when a core has data in its L1 which it has not used for a time. As other cores utilize the LLC the least recently used (LRU) algorithm may eventually evict cache lines from the LLC. As a result, the cache line in the L1 cache of the core which had previously been used will become invalidated.

### Example : A Problem
Intel Atom is capable of running many threads in parallel. As a result, the contents of the shared LLC can quickly become overwritten with new data as it is requested from memory by the cores. However, this situation highly depends on the number of simultaneous threads and their memory workloads and patterns. With the right workloads and conditions, a large portion of the LLC can be quickly overwritten with new data causing significant portions of some L1 caches to be evicted and reduce the performance of the corresponding cores.

During the time between interrupts, the low-priority processes generate memory traffic which may overwrite the entire LLC with new data and therefore invalidate everything in all other cores’ L1 caches. In this situation, the next high-priority interrupt received will see a higher latency. This is because the code and data necessary to service the instruction are no longer in the cache and must be fetched from memory.


### Solution : Intel Cache Allocation Technology
The situation described above can be alleviated using a new feature in some new Intel processors called Cache Allocation Technology (CAT). CAT does not require any modifications to the operating system or kernel to take advantage of it. By means of defining and assigning a class of service to each core, the user can assign portions of the LLC to particular cores by limiting the amount of the LLC into which each core is able to allocate cache lines. Because the core is only able to allocate cache lines into its assigned portion of the cache, it is no longer possible for the core to evict cache lines outside of this region.

The cache is divided into ways and these ways can be divided or shared among the cores with CAT. Though a core may be limited to allocating, and therefore evicting, cache lines to a subset of the LLC, a read or write from a core may still result in a cache hit if the cache line exists anywhere in the LLC.

CAT is particularly useful in scenarios where an offending application is requesting a large amount of data, putting pressure on other applications, but never reusing the data that is cached in the LLC. File hosting and video streaming programs are examples of these types of applications. On an otherwise idle system, these applications can consume the entire LLC, but never take advantage of the LLC because this offending application is not reusing most of the data that it requests. The core on which an offending application is running can be restricted to a small region of the cache. As a result, other applications have a better opportunity to benefit from the LLC but, at the same time, there is a potential for decreasing the offending application’s performance. The performance impact on the offending application depends on its workload, the size of its working set, and other factors.

Assume an application is processing data linearly on an otherwise idle system. The application has a 40MB working set and the LLC is 20MB in size. As the program processes the first 20MB of its working set, the data misses the LLC and becomes cached in the LLC after being fetched from memory. However, as the program makes requests for the next 20MB, the requests will again miss the LLC and must be serviced by memory. To make room for the next 20MB of data, the least recently used data is evicted as the following 20MB of the working set are requested. As the program continues, it thrashes the LLC as it requests new data and evicts old data in a continuous manner without hitting any of the data in the LLC.

In scenarios such as these, a user can configure the system such that the offending application is bound to a particular core(s). Additionally, the user can configure these same cores to only use, for example, ten percent of the LLC. Now the application can only thrash this ten percent of the cache, leaving the remaining ninety percent untouched.

Furthermore, in this hypothetical scenario, with the program linearly processing 40MB of data, because the application never reused the data before being thrashed in the LLC, the application’s performance is unhindered despite the reduction in the amount of the LLC into which it can allocate. This is because the performance bottleneck is still found in the memory bandwidth and latency. By limiting the cache, which the core always misses and never hits, the performance is not reduced. The cache will still miss and the memory will continue to fulfill the requests for data. However, in other scenarios where the application is more dependent on a data set which is found in the LLC (rather than RAM), utilizing CAT in this way may result in decreased performance for the application


## Cache Allocation Technology and the UP Squared board

Although the UP Squared board that we are using in the workshop does not support Cache Allocation Technology or has a shared level of cache, the MSR configuration can be practiced as it would work on an Apollo Lake-I device.

## Working with Model Specific Registers

Model Specific Registers are control registers available in x86 products to toggle some features on and off and get information about the status of e.g. the CPUs. The full list of MSRs and information to use them can be found at [Intel's Software Development Manual, Volume 3c](https://www.intel.com/content/www/us/en/architecture-and-technology/64-ia-32-architectures-software-developer-system-programming-manual-325384.html).

We are going to perform basic CAT querying and configuration. Make sure the msr module is loaded:
```$ lsmod | grep msr```

Otherwise load it with:
```$ modprobe msr```


## Reading CAT configuration from MSRs

Each CPU is mapped to a waymask. The waymask contains a bitmask where every bit enables or disables a portion of the cache to use.

In order to get the list of waymasks used by each core, type:
```c
# rdmsr -a 0xc8f
0
0
```

As mentioned before, ```0xc8f``` is the specific register where this information is stored, as described in the Software Development Manual. The switch ```-a``` outputs the information contained in the MSR for all CPUs.

The two CPUs are mapped to the first waymask. The output may vary on your system.

The waymask can be inspected with:
```c
# rdmsr -a 0xd10
```

The same command can be used to query the three remaining waymasks:

```c
# rdmsr -a 0xd11
# rdmsr -a 0xd12
# rdmsr -a 0xd13
```

The specific MSRs are described in the Software Development Manual.

## Configuring CAT via MSRs

Let's set the second way mask to include only the last two cache ways and third way mask not to include the the last two cache ways. This can be done with :
```c
# wrmsr 0xd11 0x3
# wrmsr 0xd12 0xFC
```
To see the if the configuration has been successfully written:
```c
# rdmsr -a 0xd11
CPU0: 3
CPU2: ff

# rdmsr -a 0xd12
CPU0: fc
CPU2: ff
```

To set core 0 and core 2 to second and third way mask register respectively,  execute the below command
```c
# wrmsr -p 0 0xc8f 0x100000000
# wrmsr -p 2 0xc8f 0x200000000
```
To see the if the configuration has been successfully written:
```c
# rdmsr -a 0xc8f
CPU0: 100000000
CPU2: 200000000
```
Now the core 0 has write access to the last two cache ways and core 2 has write access to the first six cache ways.

## Cache hierarchy on Linux

In order to inspect the total cache available on linux, type:
```$ lscpu | grep cache```

**To see the cache details like cache levels, cache size, cache type etc of cpu0 type :**

```$ grep '.*' /sys/devices/system/node/node*/cpu0/cache/index*/* 2>/dev/null | awk '-F[:/]' '{ printf "%6s %6s %24s %s\n" $6, $7, $9, $10, $11 ; }' ```
```c
node0  cpu0 index0      coherency_line_size 64
node0  cpu0 index0                    level 1
node0  cpu0 index0           number_of_sets 64
node0  cpu0 index0  physical_line_partition 1
node0  cpu0 index0          shared_cpu_list 0
node0  cpu0 index0           shared_cpu_map 1
node0  cpu0 index0                     size 24K
node0  cpu0 index0                     type Data
node0  cpu0 index0    ways_of_associativity 6
node0  cpu0 index1      coherency_line_size 64
node0  cpu0 index1                    level 1
node0  cpu0 index1           number_of_sets 64
node0  cpu0 index1  physical_line_partition 1
node0  cpu0 index1          shared_cpu_list 0
node0  cpu0 index1           shared_cpu_map 1
node0  cpu0 index1                     size 32K
node0  cpu0 index1                     type Instruction
node0  cpu0 index1    ways_of_associativity 8
node0  cpu0 index2      coherency_line_size 64
node0  cpu0 index2                    level 2
node0  cpu0 index2           number_of_sets 1024
node0  cpu0 index2  physical_line_partition 1
node0  cpu0 index2          shared_cpu_list 0
node0  cpu0 index2           shared_cpu_map 1
node0  cpu0 index2                     size 1024K
node0  cpu0 index2                     type Unified
node0  cpu0 index2    ways_of_associativity 16
```

**To see the details of all CPUs cache levels, cache size, cache type, no of ways, etc type :**

``` $ grep '.*' /sys/devices/system/node/node*/cpu*/cache/index*/* 2>/dev/null | awk '-F[:/]' '{ printf "%6s %6s %24s %s\n" $1, $7, $9, $10, $11 ; }' ```
