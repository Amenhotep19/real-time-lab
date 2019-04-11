# Real-Time Scheduling with PREEMPT_RT


## Introduction

The default Linux kernel includes different scheduling policies, as described in the manpage for `sched`. There are three policies relevant for real-time tasks:

* `SCHED_FIFO` implements a first-in, first-out scheduling algorithm. 
	* When a SCHED_FIFO task starts running it continues to run until either it is preempted by a higher priority thread, it is blocked by an I/O request or it calls yield function. 
	* All other tasks of lower priority will not be scheduled until SCHED_FIFO task release the CPU. 
	* Two SCHED_FIFO tasks with same priority cannot preempt each other.

* `SCHED_RR` is identical to the SCHED_FIFO scheduling, the only difference will be the way it handles the processes with the same priority. 
	* The scheduler assigns each SCHED_RR task a time slice, when the process exhausts its time slice the scheduler moves it to the end of the list of processes at its priority. 
	* In this manner, SCHED_RR task of a given priority are scheduled in round-robin amongst themselves. 
	* If there is only one process at a given priority, the RR scheduling is identical to the FIFO scheduling.

* `SCHED_DEADLINE` is implemented using Earliest Deadline First (EDF) scheduling algorithm, in conjunction with Constant Bandwidth Server (CBS). 
	* SCHED_DEADLINE policy uses three parameters to schedule tasks - Runtime, Deadline and Period. 
	* A SCHED_DEADLINE task gets "runtime" nanoseconds of CPU time for every "period" nanoseconds. The "runtime" nanoseconds should be available within "deadline" nanoseconds from the period beginning. 
	* Tasks are scheduled using EDF based on the scheduling deadlines(these are calculated every time when the task wakes up).  
	* Task with the earliest deadline is executed.
	* SCHED_DEADLINE threads are the highest priority (user controllable) threads in the system. 
	* If any SCHED_DEADLINE thread is runnable, it will preempt any thread scheduled under one of the other policies.


## Assigning Real Time Scheduling Policies to Processes

### Using chrt command
`chrt command` can be used to set the real time attributes like policy and priority of a process.
  
  Syntax to set scheduling policy to `SCHED_FIFO`
  ```
  chrt --fifo --pid <priority> <pid>
  ```
  priority values for `SCHED_FIFO` can be between 1 and 99
  
  Below example will set scheduling attribute to SCHED_FIFO for the process with pid 1823 
  ```
  root@intel-corei7-64:~# chrt --fifo --pid 99 1823  
  ```
  Syntax to set scheduling policy to `SCHED_RR`
  ```
  chrt -rr --pid <priority> <pid>
  ```
  priority values for `SCHED_RR` can be between 1 and 99
  
  Below example will set the scheduling attribute to SCHED_RR and priority 99 for the process with pid 1823 
   ```
  root@intel-corei7-64:~# chrt --rr --pid 99 1823  
  ```
  Syntax to set scheduling policy to `SCHED_DEADLINE`
  ```
  chrt --deadline --sched-runtime <nanoseconds> \
                  --sched-period <nanoseconds> \
                  --sched-deadline <nanoseconds> \
                  --pid <priority> <pid>
  ```
  priority value for `SCHED_DEADLINE` is 0.
  
  #runtime <= deadline <= period
  
 
  
  Below example will set scheduling attribute to SCHED_DEADLINE for the process with pid 1823. The runtime, deadline and period are given in nanoseconds. 
  ```
  root@intel-corei7-64:~# chrt --deadline --sched-runtime 10000 \
                                          --sched-deadline 100000 \
                                          --sched-period 1000000  \
                                          --pid 0 1823  
  ```
  
### Using system calls `sched_setscheduler` or `sched_setattr()`.

  `sched_setscheduler` function currently supporting various scheduling policies and we can use below values to set policies.
  For real-time scheduling policies we have use below values.
  1)	SCHED_FIFO
  2)	SCHED_RR

   `sched_setscheduler` can be used to set non-real-time scheduling policies like `SCHED_OTHER`, `SCHED_BATCH` and `SCHED_IDLE`. There is no support for deadline scheduling policy in sched_setscheduler function.
  
  `sched_setscheduler` function sets the scheduling policy, and priority for a thread.<br/>
   ```c
   int sched_setscheduler(pid_t pid, int policy, const struct sched_param *param);
   ```
   The real-time policies that may be specified in policy are: SCHED_FIFO, SCHED_RR
     
  Below example will set the running process with SCHED_RR scheduling with priority as 99
 
  ```c
  struct sched_param param_rr;
  memset(&param_rr, 0, sizeof(param_rr));
  param_rr.sched_priority = 99; 
  pid_t pid = getpid(); 
  if (sched_setscheduler(pid, SCHED_RR, &param_rr))
    perror("sched_setscheduler error:");
  ```
   
  Below example will set the running process with SCHED_FIFO scheduling with priority as 99
   
   ```c
   struct sched_param param_fifo;
   memset(&param_fifo, 0, sizeof(param_fifo));
   param_fifo.sched_priority = 99; 
   pid_t pid = getpid();
   if (sched_setscheduler(pid, SCHED_FIFO, &param_fifo))
     perror("sched_setscheduler error:");
   ``` 
   
   Linux supports deadline scheduling policy (SCHED_DEADLINE) from kernel version 3.14. To set SCHED_DEADLINE policy to process, we can use `sched_setattr` function.
   ```c
   int sched_setattr(pid_t pid, struct sched_attr *attr, unsigned int flags);
   ```
   
   In the below example the process in execution is assigned with the SCHED_DEADLINE policy. The process gets a runtime of 2 milliseconds for every 9 milliseconds period. The runtime milliseconds should be available within 5 milliseconds of deadline from the period beginning.
   
   ```c
   #define _GNU_SOURCE
   #include <stdint.h>
   #include <stdio.h>
   #include <unistd.h>
   #include <sys/syscall.h>
   #include <sched.h>
   #include <string.h>
   #include <linux/sched.h>
   #include <sys/types.h>

   struct sched_attr {
    uint32_t size;
    uint32_t sched_policy;
    uint64_t sched_flags;
    int32_t sched_nice;
    uint32_t sched_priority;
    uint64_t sched_runtime;
    uint64_t sched_deadline;
    uint64_t sched_period;
  };
  
  int sched_setattr(pid_t pid, const struct sched_attr *attr, unsigned int flags) {
     return syscall(__NR_sched_setattr, pid, attr, flags); 
  }
  
  int main() {
	   unsigned int flags = 0;
	   int status = -1;
	   struct sched_attr attr_deadline;
	   memset(&attr_deadline, 0, sizeof(attr_deadline));
	   pid_t pid =  getpid();
	   attr_deadline.sched_policy = SCHED_DEADLINE;
	   attr_deadline.sched_runtime = 2*1000*1000;
	   attr_deadline.sched_deadline = 5*1000*1000;
	   attr_deadline.sched_period = 9*1000*1000;
	   attr_deadline.size = sizeof(attr_deadline);
	   attr_deadline.sched_flags = 0;
	   attr_deadline.sched_nice = 0;
	   attr_deadline.sched_priority = 0;
	   status = sched_setattr(pid,&attr_deadline,flags)
	   if(status)
	   	perror("sched_setscheduler error:");
	   return 0;
  }
  ```
  
  ### Using `pthread` functions
   The scheduling policy for threads can be set using the pthread functions `pthread_attr_setschedpolicy`, `pthread_attr_setschedparam`, `pthread_attr_setinheritsched`. The below steps will give a understanding of creating a thread using FIFO scheduling policy using pthread functions.
   
   To create a thread using FIFO scheduling initialize the `pthread_attr_t`(thread attribute object) object using `pthread_attr_init` function.
  ```c
  pthread_attr_t attr_fifo;
  pthread_attr_init(&attr_fifo) ;
  ``` 
  
  After initialization, set the thread attributes object referred to by attr_fifo to SCHED_FIFO(FIFO scheduling policy) using `pthread_attr_setschedpolicy`. 
  ```c
  pthread_attr_setschedpolicy(&attr_fifo, SCHED_FIFO);
  ```

  Set the priority (can take values between 1-99 for FIFO scheduling) of the thread using the sched_param object and copy the param values to thread attribute using `pthread_attr_setschedparam`. 
  ```c
  struct sched_param param_fifo;
  param_fifo.sched_priority = 92;
  pthread_attr_setschedparam(&attr_fifo, &param_fifo);
  ```
  
  Set the inherit-scheduler attribute of the thread attribute. The inherit-scheduler attribute determines if new thread takes scheduling attributes from the calling thread or from the attr. To use the scheduling attribute used in attr call the function `pthread_attr_setinheritsched` using `PTHREAD_EXPLICIT_SCHED`.
  ```c
  pthread_attr_setinheritsched(&attr_fifo, PTHREAD_EXPLICIT_SCHED);
  ```
  
  In the next step create the thread by calling `pthread_create` function
  ```c
  pthread_t thread_fifo;
  pthread_create(&thread_fifo, &attr_fifo, thread_function_fifo, NULL);
  ```
  
  The new thread is created with FIFO scheduling scheme.
  
  Find the complete example below
  ```c
   #include <pthread.h>
   #include <stdio.h>
   
   void *thread_function_fifo(void *data) {
   	printf("Inside Thread\n");
   	return NULL;
	}
	
	int main(int argc, char* argv[]) {
		struct sched_param param_fifo;
		pthread_attr_t attr_fifo;
		pthread_t thread_fifo;
		int status = -1;
		memset(&param_fifo, 0, sizeof(param_fifo));
		status = pthread_attr_init(&attr_fifo);
		if (status) {
			printf("pthread_attr_init failed\n");
			return status;
		}
		status = pthread_attr_setschedpolicy(&attr_fifo, SCHED_FIFO);
		if (status) {
			printf("pthread_attr_setschedpolicy failed\n");
			return status;
		}
		param_fifo.sched_priority = 92;
		status = pthread_attr_setschedparam(&attr_fifo, &param_fifo);
		if (status) {
			printf("pthread_attr_setschedparam failed\n");
			return status;
		}
		status = pthread_attr_setinheritsched(&attr_fifo, PTHREAD_EXPLICIT_SCHED);
		if (status) {
			printf("pthread_attr_setinheritsched failed\n");
			return status;
		}
		status = pthread_create(&thread_fifo, &attr_fifo, thread_function_fifo, NULL);
		if (status) {
			printf("pthread_create failed\n");
			return status;
		}
		pthread_join(thread_fifo, NULL);
		return status;
	}
   ```
   
   
## Inspecting Scheduling Configuration

The policy of the running processes and threads can be inspected by using ps and querying for the attribute "policy". It is aliased as "class or cls" as well, and will provide for example
* TS: for regular time-sharing scheduler (SCHED_OTHER in POSIX 1b)
* FF: POSIX 1b FIFO
* RR: POSIX 1b Round Robin
* #6: deadline (#6 is hard coded integer for SCHED_DEADLINE in kernel sched.h)

The attribute for the real-time priority is ```rtprio``` and will display the priority associated to that process or thread.


### Processes

For example, we can get a tree of all the processes executing on the computer, listed together with theír Process ID, Scheduling Policy, Real-Time Priority and Command Line by running:

```
root@intel-corei7-64:~# ps f -g 0 -o pid,policy,rtprio,cmd
 PID POL RTPRIO CMD
    2 TS       - [kthreadd]
    3 TS       -  \_ [ksoftirqd/0]
    4 FF       1  \_ [ktimersoftd/0]
    6 TS       -  \_ [kworker/0:0H]
    8 FF       1  \_ [rcu_preempt]
    9 FF       1  \_ [rcu_sched]
   10 FF       1  \_ [rcub/0]
   11 FF       1  \_ [rcuc/0]
   12 TS       -  \_ [kswork]
   13 FF      99  \_ [posixcputmr/0]
   14 FF      99  \_ [migration/0]
   15 TS       -  \_ [cpuhp/0]
   16 TS       -  \_ [cpuhp/2]
   17 FF      99  \_ [migration/2]
   18 FF       1  \_ [rcuc/2]
   19 FF       1  \_ [ktimersoftd/2]
   20 TS       -  \_ [ksoftirqd/2]
   22 TS       -  \_ [kworker/2:0H]
   23 FF      99  \_ [posixcputmr/2]
   24 TS       -  \_ [kdevtmpfs]
   25 TS       -  \_ [netns]
   27 TS       -  \_ [oom_reaper]
   28 TS       -  \_ [writeback]
   29 TS       -  \_ [kcompactd0]
   30 TS       -  \_ [crypto]
   31 TS       -  \_ [bioset]
   32 TS       -  \_ [kblockd]
   33 FF      50  \_ [irq/9-acpi]
   34 TS       -  \_ [md]
   35 TS       -  \_ [watchdogd]
   36 TS       -  \_ [rpciod]
   37 TS       -  \_ [xprtiod]
   39 TS       -  \_ [kswapd0]
   40 TS       -  \_ [vmstat]
   41 TS       -  \_ [nfsiod]
   63 TS       -  \_ [kthrotld]
   66 TS       -  \_ [bioset]
   67 TS       -  \_ [bioset]
   68 TS       -  \_ [bioset]
   69 TS       -  \_ [bioset]
   70 TS       -  \_ [bioset]
   71 TS       -  \_ [bioset]
   72 TS       -  \_ [bioset]
   73 TS       -  \_ [bioset]
   74 TS       -  \_ [bioset]
   75 TS       -  \_ [bioset]
   76 TS       -  \_ [bioset]
   77 TS       -  \_ [bioset]
   78 TS       -  \_ [bioset]
   79 TS       -  \_ [bioset]
   80 TS       -  \_ [bioset]
   81 TS       -  \_ [bioset]
   83 TS       -  \_ [bioset]
   84 TS       -  \_ [bioset]
   85 TS       -  \_ [bioset]
   86 TS       -  \_ [bioset]
   87 TS       -  \_ [bioset]
   88 TS       -  \_ [bioset]
   89 TS       -  \_ [bioset]
   90 TS       -  \_ [bioset]
   92 FF      50  \_ [irq/27-idma64.0]
   93 FF      50  \_ [irq/27-i2c_desi]
   94 FF      50  \_ [irq/28-idma64.1]
   95 FF      50  \_ [irq/28-i2c_desi]
   96 FF      50  \_ [irq/29-idma64.2]
   98 FF      50  \_ [irq/29-i2c_desi]
  100 FF      50  \_ [irq/30-idma64.3]
  101 FF      50  \_ [irq/30-i2c_desi]
  102 FF      50  \_ [irq/31-idma64.4]
  103 FF      50  \_ [irq/31-i2c_desi]
  104 FF      50  \_ [irq/32-idma64.5]
  106 FF      50  \_ [irq/32-i2c_desi]
  107 FF      50  \_ [irq/33-idma64.6]
  108 FF      50  \_ [irq/33-i2c_desi]
  109 FF      50  \_ [irq/34-idma64.7]
  110 FF      50  \_ [irq/34-i2c_desi]
  111 FF      50  \_ [irq/4-idma64.8]
  112 FF      50  \_ [irq/5-idma64.9]
  113 FF      50  \_ [irq/35-idma64.1]
  115 FF      50  \_ [irq/37-idma64.1]
  117 TS       -  \_ [nvme]
  118 FF      50  \_ [irq/365-xhci_hc]
  121 TS       -  \_ [scsi_eh_0]
  122 TS       -  \_ [scsi_tmf_0]
  123 TS       -  \_ [usb-storage]
  124 TS       -  \_ [dm_bufio_cache]
  125 FF      50  \_ [irq/39-mmc0]
  126 FF      49  \_ [irq/39-s-mmc0]
  127 FF      50  \_ [irq/42-mmc1]
  128 FF      49  \_ [irq/42-s-mmc1]
  129 TS       -  \_ [ipv6_addrconf]
  144 TS       -  \_ [bioset]
  175 TS       -  \_ [bioset]
  177 TS       -  \_ [mmcqd/0]
  182 TS       -  \_ [bioset]
  187 TS       -  \_ [mmcqd/0boot0]
  189 TS       -  \_ [bioset]
  191 TS       -  \_ [mmcqd/0boot1]
  193 TS       -  \_ [bioset]
  195 TS       -  \_ [mmcqd/0rpmb]
  332 TS       -  \_ [kworker/2:1H]
  342 TS       -  \_ [kworker/0:1H]
  407 TS       -  \_ [jbd2/mmcblk0p2-]
  409 TS       -  \_ [ext4-rsv-conver]
  429 TS       -  \_ [bioset]
  463 TS       -  \_ [loop0]
  466 TS       -  \_ [jbd2/loop0-8]
  467 TS       -  \_ [ext4-rsv-conver]
  558 FF      50  \_ [irq/366-mei_me]
  559 FF      50  \_ [irq/8-rtc0]
  560 FF      50  \_ [irq/35-pxa2xx-s]
  561 TS       -  \_ [spi1]
  563 FF      50  \_ [irq/37-pxa2xx-s]
  564 TS       -  \_ [spi3]
  572 FF      50  \_ [irq/369-i915]
  798 FF       1  \_ [i915/signal:0]
  800 FF       1  \_ [i915/signal:1]
  801 FF       1  \_ [i915/signal:2]
  802 FF       1  \_ [i915/signal:4]
  832 FF      50  \_ [irq/367-enp2s0]
  835 FF      50  \_ [irq/368-enp3s0]
  844 FF      50  \_ [irq/4-serial]
  846 FF      50  \_ [irq/5-serial]
 4167 TS       -  \_ [kworker/0:1]
 4194 TS       -  \_ [kworker/u8:1]
 4234 TS       -  \_ [kworker/2:0]
 4242 TS       -  \_ [kworker/0:0]
 4288 TS       -  \_ [kworker/u8:0]
 4313 TS       -  \_ [kworker/2:1]
 4318 TS       -  \_ [kworker/0:2]
    1 TS       - /sbin/init initrd=\initrd LABEL=boot processor.max_cstate=0 intel_idle.max_cstate=0 clocksource=tsc tsc=reliable nmi_watchdog=0 nosoftlockup intel_pstate=disable i915.disable_power_well=0 i915.enable_rc6=0 idle=poll noht 3 snd_hda_intel.power_save=1 snd_hda_intel.power_save_controller=y scsi_mod.scan=async console=ttyS2,115200 rootwait console=ttyS0,115200 console=tty0
  491 TS       - /lib/systemd/systemd-journald
  525 TS       - /lib/systemd/systemd-udevd
  551 TS       - /usr/sbin/syslog-ng -F -p /var/run/syslogd.pid
  571 TS       - /usr/sbin/jhid -d
  625 TS       - /usr/sbin/connmand -n
  629 TS       - /lib/systemd/systemd-logind
  631 TS       - /usr/sbin/thermald --no-daemon --dbus-enable
  658 TS       - /usr/sbin/ofonod -n
  712 TS       - /usr/sbin/acpid
  828 TS       - /sbin/agetty --noclear tty1 linux
  829 TS       - /sbin/agetty -8 -L ttyS0 115200 xterm
  830 TS       - /sbin/agetty -8 -L ttyS1 115200 xterm
  834 TS       - /usr/sbin/wpa_supplicant -u
  836 TS       - /usr/sbin/nmbd
  849 TS       - /usr/sbin/smbd
  850 TS       -  \_ /usr/sbin/smbd
  851 TS       -  \_ /usr/sbin/smbd
  853 TS       -  \_ /usr/sbin/smbd
  855 TS       - /usr/sbin/dropbear -i -r /etc/dropbear/dropbear_rsa_host_key -B
  856 TS       -  \_ -sh
 3804 TS       - /usr/sbin/dropbear -i -r /etc/dropbear/dropbear_rsa_host_key -B
 3805 TS       -  \_ -sh
 4394 TS       -  \_ ps f -g 0 -o pid,policy,rtprio,cmd
 4393 TS       - /sbin/agetty -8 -L ttyS2 115200 xterm
```

As usual, the processes between brackets belong to the kernel. In this first example we can already see processes with regular scheduling policy (timesharing, ```TS```) coexisting with processes with a real time policy (FIFO, ```FF```). What is more interesting is that we can already see some priorities in action:

```
 PID POL RTPRIO CMD
    2 TS       - [kthreadd]
    4 FF       1  \_ [ktimersoftd/0]
    8 FF       1  \_ [rcu_preempt]
    9 FF       1  \_ [rcu_sched]
   10 FF       1  \_ [rcub/0]
   11 FF       1  \_ [rcuc/0]
   13 FF      99  \_ [posixcputmr/0]
   14 FF      99  \_ [migration/0]
   17 FF      99  \_ [migration/2]
   18 FF       1  \_ [rcuc/2]
   19 FF       1  \_ [ktimersoftd/2]
   23 FF      99  \_ [posixcputmr/2]
   33 FF      50  \_ [irq/9-acpi]
   92 FF      50  \_ [irq/27-idma64.0]
   93 FF      50  \_ [irq/27-i2c_desi]
   94 FF      50  \_ [irq/28-idma64.1]
   95 FF      50  \_ [irq/28-i2c_desi]
   96 FF      50  \_ [irq/29-idma64.2]
   98 FF      50  \_ [irq/29-i2c_desi]
  100 FF      50  \_ [irq/30-idma64.3]
  101 FF      50  \_ [irq/30-i2c_desi]
  102 FF      50  \_ [irq/31-idma64.4]
  103 FF      50  \_ [irq/31-i2c_desi]
  104 FF      50  \_ [irq/32-idma64.5]
  106 FF      50  \_ [irq/32-i2c_desi]
  107 FF      50  \_ [irq/33-idma64.6]
  108 FF      50  \_ [irq/33-i2c_desi]
  109 FF      50  \_ [irq/34-idma64.7]
  110 FF      50  \_ [irq/34-i2c_desi]
  111 FF      50  \_ [irq/4-idma64.8]
  112 FF      50  \_ [irq/5-idma64.9]
  113 FF      50  \_ [irq/35-idma64.1]
  115 FF      50  \_ [irq/37-idma64.1]
  118 FF      50  \_ [irq/365-xhci_hc]
  125 FF      50  \_ [irq/39-mmc0]
  126 FF      49  \_ [irq/39-s-mmc0]
  127 FF      50  \_ [irq/42-mmc1]
  128 FF      49  \_ [irq/42-s-mmc1]
  558 FF      50  \_ [irq/366-mei_me]
  559 FF      50  \_ [irq/8-rtc0]
  560 FF      50  \_ [irq/35-pxa2xx-s]
  563 FF      50  \_ [irq/37-pxa2xx-s]
  572 FF      50  \_ [irq/369-i915]
  798 FF       1  \_ [i915/signal:0]
  800 FF       1  \_ [i915/signal:1]
  801 FF       1  \_ [i915/signal:2]
  802 FF       1  \_ [i915/signal:4]
  832 FF      50  \_ [irq/367-enp2s0]
  835 FF      50  \_ [irq/368-enp3s0]
  844 FF      50  \_ [irq/4-serial]
  846 FF      50  \_ [irq/5-serial]
```
Example for setting a deadline policy to a task

 Execute below command to know the current policy of a task

  ```
  root@intel-corei7-64:~# ps f -g 0 -o pid,policy,rtprio,cmd
  PID POL RTPRIO CMD
     1 TS       - /sbin/init nosoftlockup noht 3
   185 TS       - /lib/systemd/systemd-journald
   209 TS       - /lib/systemd/systemd-udevd
   472 RR      99 /usr/sbin/acpid
   476 TS       - /usr/sbin/thermald --no-daemon --dbus-enable
   486 TS       - /usr/sbin/jhid -d
  ```
  Execute below command to change the policy to SCHED_DEADLINE(#6)

  ```
  chrt --deadline --sched-runtime 10000 --sched-deadline 100000 --sched-period 1000000 -p 0 472
  ```
  Execute below command to see change in the policy of a task
  
  ```
   root@intel-corei7-64:~# ps f -g 0 -o pid,policy,rtprio,cmd
   PID POL RTPRIO CMD
      1 TS       - /sbin/init nosoftlockup noht 3
    185 TS       - /lib/systemd/systemd-journald
    209 TS       - /lib/systemd/systemd-udevd
    472 #6       0 /usr/sbin/acpid
    476 TS       - /usr/sbin/thermald --no-daemon --dbus-enable
    486 TS       - /usr/sbin/jhid -d
   ```
   
Condensing this information in a table:

| Prio | Names |
| ---- | ----- |
| 99  | posixcputmr, migration|
| 50 | All IRQ handlers except 39-s-mmc0 and 42-s-mmc1. E.g. 367-enp2s0 deals with one of the network interfaces |
| 49  | IRQ handlers 39-s-mmc0 and 42-s-mmc1 |
| 1  | i915/signal, ktimersoftd, rcu_preempt, rcu_sched, rcub, rcuc|
| 0 | The rest of the tasks currently running |

The highest priority real-time tasks in this system are the timers and migration threads, with prio 99. The lowest priority real-time tasks are FIXME:check what ktimersoftd does. But also remember that there are many timesharing tasks including the kernel ones that will run once there are no more FIFO tasks pending.
Most of the IRQ handlers are running with prio 50.



### Threads

It is important to note that the listing above does not list userspace threads. In order to get that information, we can use the following command, that displays the number of threads for the Process ID (Number of LightWeight Processes, ```NLWP```), and the Thread ID (```TID```):

```
root@intel-corei7-64:~# ps -eTo pid,nlwp,tid,policy,rtprio,cmd
  PID NLWP   TID POL RTPRIO CMD
    1    1     1 TS       - /sbin/init initrd=\initrd LABEL=boot processor.max_cstate=0 intel_idle.max_cstate=0 clocksource=tsc tsc=reliable nmi_watchdog=0 nosoftlockup intel_pstate=disable i915.disable_power_well=0 i915.enable_rc6=0 idle=poll noht 3 snd_hda_intel.power_save=1 snd_hda_intel.power_save_controller=y scsi_mod.scan=async console=ttyS2,115200 rootwait console=ttyS0,115200 console=tty0
    2    1     2 TS       - [kthreadd]
    3    1     3 TS       - [ksoftirqd/0]
    4    1     4 FF       1 [ktimersoftd/0]
    6    1     6 TS       - [kworker/0:0H]
    8    1     8 FF       1 [rcu_preempt]
    9    1     9 FF       1 [rcu_sched]
   10    1    10 FF       1 [rcub/0]
   11    1    11 FF       1 [rcuc/0]
   12    1    12 TS       - [kswork]
   13    1    13 FF      99 [posixcputmr/0]
   14    1    14 FF      99 [migration/0]
   15    1    15 TS       - [cpuhp/0]
   16    1    16 TS       - [cpuhp/2]
   17    1    17 FF      99 [migration/2]
   18    1    18 FF       1 [rcuc/2]
   19    1    19 FF       1 [ktimersoftd/2]
   20    1    20 TS       - [ksoftirqd/2]
   22    1    22 TS       - [kworker/2:0H]
   23    1    23 FF      99 [posixcputmr/2]
   24    1    24 TS       - [kdevtmpfs]
   25    1    25 TS       - [netns]
   27    1    27 TS       - [oom_reaper]
   28    1    28 TS       - [writeback]
   29    1    29 TS       - [kcompactd0]
   30    1    30 TS       - [crypto]
   31    1    31 TS       - [bioset]
   32    1    32 TS       - [kblockd]
   33    1    33 FF      50 [irq/9-acpi]
   34    1    34 TS       - [md]
   35    1    35 TS       - [watchdogd]
   36    1    36 TS       - [rpciod]
   37    1    37 TS       - [xprtiod]
   39    1    39 TS       - [kswapd0]
   40    1    40 TS       - [vmstat]
   41    1    41 TS       - [nfsiod]
   63    1    63 TS       - [kthrotld]
   66    1    66 TS       - [bioset]
   67    1    67 TS       - [bioset]
   68    1    68 TS       - [bioset]
   69    1    69 TS       - [bioset]
   70    1    70 TS       - [bioset]
   71    1    71 TS       - [bioset]
   72    1    72 TS       - [bioset]
   73    1    73 TS       - [bioset]
   74    1    74 TS       - [bioset]
   75    1    75 TS       - [bioset]
   76    1    76 TS       - [bioset]
   77    1    77 TS       - [bioset]
   78    1    78 TS       - [bioset]
   79    1    79 TS       - [bioset]
   80    1    80 TS       - [bioset]
   81    1    81 TS       - [bioset]
   83    1    83 TS       - [bioset]
   84    1    84 TS       - [bioset]
   85    1    85 TS       - [bioset]
   86    1    86 TS       - [bioset]
   87    1    87 TS       - [bioset]
   88    1    88 TS       - [bioset]
   89    1    89 TS       - [bioset]
   90    1    90 TS       - [bioset]
   92    1    92 FF      50 [irq/27-idma64.0]
   93    1    93 FF      50 [irq/27-i2c_desi]
   94    1    94 FF      50 [irq/28-idma64.1]
   95    1    95 FF      50 [irq/28-i2c_desi]
   96    1    96 FF      50 [irq/29-idma64.2]
   98    1    98 FF      50 [irq/29-i2c_desi]
  100    1   100 FF      50 [irq/30-idma64.3]
  101    1   101 FF      50 [irq/30-i2c_desi]
  102    1   102 FF      50 [irq/31-idma64.4]
  103    1   103 FF      50 [irq/31-i2c_desi]
  104    1   104 FF      50 [irq/32-idma64.5]
  106    1   106 FF      50 [irq/32-i2c_desi]
  107    1   107 FF      50 [irq/33-idma64.6]
  108    1   108 FF      50 [irq/33-i2c_desi]
  109    1   109 FF      50 [irq/34-idma64.7]
  110    1   110 FF      50 [irq/34-i2c_desi]
  111    1   111 FF      50 [irq/4-idma64.8]
  112    1   112 FF      50 [irq/5-idma64.9]
  113    1   113 FF      50 [irq/35-idma64.1]
  115    1   115 FF      50 [irq/37-idma64.1]
  117    1   117 TS       - [nvme]
  118    1   118 FF      50 [irq/365-xhci_hc]
  121    1   121 TS       - [scsi_eh_0]
  122    1   122 TS       - [scsi_tmf_0]
  123    1   123 TS       - [usb-storage]
  124    1   124 TS       - [dm_bufio_cache]
  125    1   125 FF      50 [irq/39-mmc0]
  126    1   126 FF      49 [irq/39-s-mmc0]
  127    1   127 FF      50 [irq/42-mmc1]
  128    1   128 FF      49 [irq/42-s-mmc1]
  129    1   129 TS       - [ipv6_addrconf]
  144    1   144 TS       - [bioset]
  175    1   175 TS       - [bioset]
  177    1   177 TS       - [mmcqd/0]
  182    1   182 TS       - [bioset]
  187    1   187 TS       - [mmcqd/0boot0]
  189    1   189 TS       - [bioset]
  191    1   191 TS       - [mmcqd/0boot1]
  193    1   193 TS       - [bioset]
  195    1   195 TS       - [mmcqd/0rpmb]
  332    1   332 TS       - [kworker/2:1H]
  342    1   342 TS       - [kworker/0:1H]
  407    1   407 TS       - [jbd2/mmcblk0p2-]
  409    1   409 TS       - [ext4-rsv-conver]
  429    1   429 TS       - [bioset]
  463    1   463 TS       - [loop0]
  466    1   466 TS       - [jbd2/loop0-8]
  467    1   467 TS       - [ext4-rsv-conver]
  491    1   491 TS       - /lib/systemd/systemd-journald
  525    1   525 TS       - /lib/systemd/systemd-udevd
  533    2   533 TS       - /lib/systemd/systemd-timesyncd
  533    2   548 TS       - /lib/systemd/systemd-timesyncd
  551    1   551 TS       - /usr/sbin/syslog-ng -F -p /var/run/syslogd.pid
  553    1   553 TS       - /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
  558    1   558 FF      50 [irq/366-mei_me]
  559    1   559 FF      50 [irq/8-rtc0]
  560    1   560 FF      50 [irq/35-pxa2xx-s]
  561    1   561 TS       - [spi1]
  563    1   563 FF      50 [irq/37-pxa2xx-s]
  564    1   564 TS       - [spi3]
  571    1   571 TS       - /usr/sbin/jhid -d
  572    1   572 FF      50 [irq/369-i915]
  625    1   625 TS       - /usr/sbin/connmand -n
  626    1   626 TS       - avahi-daemon: running [intel-corei7-64.local]
  629    1   629 TS       - /lib/systemd/systemd-logind
  631    2   631 TS       - /usr/sbin/thermald --no-daemon --dbus-enable
  631    2   819 TS       - /usr/sbin/thermald --no-daemon --dbus-enable
  658    1   658 TS       - /usr/sbin/ofonod -n
  712    1   712 TS       - /usr/sbin/acpid
  770    1   770 TS       - /usr/sbin/rpcbind
  781    1   781 TS       - avahi-daemon: chroot helper
  798    1   798 FF       1 [i915/signal:0]
  800    1   800 FF       1 [i915/signal:1]
  801    1   801 FF       1 [i915/signal:2]
  802    1   802 FF       1 [i915/signal:4]
  821    1   821 TS       - /usr/sbin/rpc.statd -F
  828    1   828 TS       - /sbin/agetty --noclear tty1 linux
  829    1   829 TS       - /sbin/agetty -8 -L ttyS0 115200 xterm
  830    1   830 TS       - /sbin/agetty -8 -L ttyS1 115200 xterm
  832    1   832 FF      50 [irq/367-enp2s0]
  834    1   834 TS       - /usr/sbin/wpa_supplicant -u
  835    1   835 FF      50 [irq/368-enp3s0]
  836    1   836 TS       - /usr/sbin/nmbd
  844    1   844 FF      50 [irq/4-serial]
  846    1   846 FF      50 [irq/5-serial]
  849    1   849 TS       - /usr/sbin/smbd
  850    1   850 TS       - /usr/sbin/smbd
  851    1   851 TS       - /usr/sbin/smbd
  853    1   853 TS       - /usr/sbin/smbd
  855    1   855 TS       - /usr/sbin/dropbear -i -r /etc/dropbear/dropbear_rsa_host_key -B
  856    1   856 TS       - -sh
 3804    1  3804 TS       - /usr/sbin/dropbear -i -r /etc/dropbear/dropbear_rsa_host_key -B
 3805    1  3805 TS       - -sh
 4194    1  4194 TS       - [kworker/u8:1]
 4234    1  4234 TS       - [kworker/2:0]
 4242    1  4242 TS       - [kworker/0:0]
 4288    1  4288 TS       - [kworker/u8:0]
 4313    1  4313 TS       - [kworker/2:1]
 4318    1  4318 TS       - [kworker/0:2]
 4401    1  4401 TS       - [kworker/0:1]
 4404    1  4404 TS       - [kworker/2:2]
 4448    1  4448 TS       - [kworker/u8:2]
 4453    1  4453 TS       - /sbin/agetty -8 -L ttyS2 115200 xterm
 4454    1  4454 TS       - ps -eTo pid,nlwp,tid,policy,rtprio,cmd
```

For example ```systemd-timesyncd``` has two threads:
```
 533    2   533 TS       - /lib/systemd/systemd-timesyncd
 533    2   548 TS       - /lib/systemd/systemd-timesyncd
```


## Using cyclictest with priorities

Now let’s go back to cyclictest.

In order to compare different runs, we are going to use a parameter to specify a fixed number of iterations:

```
root@intel-corei7-64:~# cyclictest -l 10000
# /dev/cpu_dma_latency set to 0us
policy: other/other: loadavg: 0.05 0.01 0.00 1/165 5520          

T: 0 ( 5518) P: 0 I:1000 C:  10000 Min:      7 Act:   21 Avg:   23 Max:     460
```
Cyclictest is supporting below scheduler policies 

* ```other```    - SCHED_OTHER is the most widely used policy. These tasks do not have static priorities. Instead they have a "nice" value ranging from -20 (highest) to +19 (lowest). This scheduling policy is quite different from the real-time policies in that the scheduler aims at a "fair" distribution of the CPU.
* ```normal```   - also called SCHED_OTHER. 
*	```batch```    - SCHED_BATCH is very similar to SCHED_OTHER. The difference is that SCHED_BATCH is optimized for throughput.
*	```idle```     - SCHED_IDLE is similar to SCHED_OTHER. 
*	```fifo```     - SCHED_FIFO tasks are allowed to run until they have completed their work or voluntarily yields.
*	```rr```       - SCHED_RR tasks are allowed to run until they have completed their work,   until they voluntarily yields, or until they have consumed a specified amount of CPU time.


Now we can run it using the FIFO policy:

```
root@intel-corei7-64:~# cyclictest --loops=10000 --policy=fifo
defaulting realtime priority to 2
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.29 0.06 0.02 1/165 5561          

T: 0 ( 5559) P: 2 I:1000 C:  10000 Min:      9 Act:   11 Avg:   21 Max:     108
```

And set the priority:

```
root@intel-corei7-64:~# cyclictest --loops=10000 --policy=fifo --priority=98
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.34 0.10 0.03 1/165 5573          

T: 0 ( 5570) P:98 I:1000 C:  10000 Min:     18 Act:   24 Avg:   26 Max:      84
```
## Running cyclictest beside different workloads

The cyclictest test is used for bench-marking real-time systems, by measuring the latency. In the below examples we are generating different workloads beside cyclictest to ```benchmark``` a real-time system.

Example for generating CPU workloads

Running in process mode with 15 groups using 30 file descriptors and each sender will pass 100 messages of 256 byes. Use below code to generate workload and run cyclictest parallel in other terminal to observe the latency. 

Following code is a way to generate CPU load.
```c
#!/bin/sh

#For CPU Load 
count=1;
while [[ count -le 5 ]];
do 
  hackbench -s 256 -l 100 -g 15 -f 15 ;
  count=$((count+1));
  sleep 5;
done;
```
Run below command to know the latency
```
root@intel-corei7-64:~# cyclictest -l 10000  //run command in new terminal.
# /dev/cpu_dma_latency set to 0us
policy: other/other: loadavg: 20.45 5.45 1.86 1/169 3810
T: 0 ( 2903) P: 0 I:1000 C:  10000 Min:      7 Act:   22 Avg:   25 Max:    4138          
```
Example for Generating I/O workloads 

Taskset will launch ```du command``` with a given CPU affinity, du estimates and displays the disk space used by files from root directory.

```du command``` operation involves heavy access to the filesystem.

Use below command to generate I/O workload

```
root@intel-corei7-64:~# taskset -c 2 du /
```
Run below command to know the latency.
```
root@intel-corei7-64:~# cyclictest -l 10000  //run command in new terminal
# /dev/cpu_dma_latency set to 0us
policy: other/other: loadavg: 0.10 0.28 0.15 1/159 872          
T: 0 (  869) P: 0 I:1000 C:  10000 Min:      7 Act:   22 Avg:   24 Max:    1998
```

## ```cyclictest``` based benchmark with and without neighboring stress 

The `rt_bmark` script is used to run the cyclictest based benchmark with neighboring stress. The script is used to capture the interrupt latency by varying the system load. This behavior is attained using cyclictest and stress test tools.

The script will measure latency with different neighboring stress. It uses six types of load for stress test in each pass - `"no stress", "io", "cpu", "hdd", "vm" and "full"(all the four stress parameters are applied)`.

`stress` test is performed by script using following command. 

```
root@intel-corei7-64:~# stress -i 2 -c 2 -d 2 --hdd-bytes 20M -m 2 --vm-bytes 10M
```

Where -i or --io create N worker threads to perform I/O sync <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -c or --cpu create N worker threads to perform sqrt operations <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; --hdd-bytes write specified bytes for each worker thread <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; --vm-bytes allocate specified bytes for each workerer thread using malloc <br/>

The above command is run only during the iteration when all the parameters of stress are applied. During the individual stress test only respective stress parameter is applied.
	  
The `rt_bmark` script runs cyclictest to measure the latency with the below options
```
root@intel-corei7-64:~# cyclictest -S -p 99 -q -i 100 -d 20 -l 30000
```
Where -S signifies testing on SMP systems. The -S will include -a, -t and -n options <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -a Run the threads on specified processor cores <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -n Use the clock_nanosleep <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -t number of test threads, if not specified will be equal to number of CPU's <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -p set the priority of the first threads <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -q prints only summary output <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -i set interval of thread in microseconds <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -d distance of thread in microseconds <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -l no of loops <br/>

The `rt_bmark` captures minimum, maximum and average latency values for each type of cyclic and stress test combination.


## Interrupts

`Interrupts` can be described as "immediate response to hardware events", every ISR (Interrupt Service Routine) can expect some delay or latency. Based on what is causing those latencies the latencies are divided into two components - `Hardware Interrupt Latency` and `Software Interrupt Latency`.

`Hardware interrupt` latency reflects the time required for operations like instruction in execution completion in the CPU, locating address of interrupt handler, saving all of the registers data.

`Software latency` can be predicted based on the system interrupt-disable time and the size of the system ISR (Interrupt Service Routine) prologue. This is the code that saves the registers manually, it also performs operations before the start of interrupt handler.

### Different Types of Interrupts

*	`Legacy interrupts` – These are also called as fixed interrupts they use older bus technologies in this the interrupts are signaled using one or more external pins which are wired as "out-of-band" i.e. which are separate from the main line of the bus.
*	`Message-signaled interrupts` – MSI use in-band messages which target addresses in host bridge. These send data along with interrupt message since these are unshared messages these are unique within the system. A PCI (Peripheral Component Interconnect) function request up to 32 MSI messages.
*	`Extended message-signaled interrupts` – These are an enhanced version of MSIs. The MSI-X support 2048 messages with separate address and data fields for each message.
  
## Using MSI latency test
Now evaluates MSI latency from an i210 PCIe device interrupt source, identified as following 
```
root@intel-corei7-64:~# ethtool -i enp2s0
driver: igb_avb
version: 5.3.2_AVB
firmware-version:  0. 6-5
expansion-rom-version:
bus-info: 0000:02:00.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: no
```
Simply run the test at kernel-level replacing the loaded module by the ``` msi_lat.ko``` and dump report in kernel log message. 
```
root@intel-corei7-64:~# /bin/echo -n 0000:02:00.0 > /sys/bus/pci/drivers/igb_avb/unbind
root@intel-corei7-64:~# echo '/bin/dmesg -c  &> /dev/null; \
                        /sbin/insmod ~/msilatency/msi_lat.ko core=1 && \
			sleep 26 && /sbin/rmmod msi_lat; \
			/bin/dmesg > /tmp/result_msi_lat.txt
root@intel-corei7-64:~# /bin/echo -n 0000:02:00.0 > /sys/bus/pci/drivers/igb_avb/bind
```
### System Management Interrupts (SMI)

```System Management Interrupts``` can be used to report hardware errors, power capping, external policies, thermal throttling, etc. CPU enters System Management Mode (SMM) when SMI is received and a handler is executed to handle the SMI. System management firmware like BIOS or EFI will provide the SMM directly.

SMI has the highest priority than all the other interrupts.

A poorly written SMI handler can consume many milliseconds of CPU time which increases latency hence SMI handlers should be written carefully.

To measure the preemption latency `cyclictest` is invoked to create 100000 real-time threads with priority 90 and scheduling policy as SCHED_FIFO. 

```
root@intel-corei7-64:~# cyclictest -l100000 -m -Sp90 -i200 -h400
```

The below histogram is plotted between number of threads and latency in microseconds.

![1](https://github.intel.com/storage/user/9783/files/e8d3c2c0-a189-11e8-8023-0c1aab276b79)
