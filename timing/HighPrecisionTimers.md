# High Precision Timers
Every application looks for completing its task at exact defined time. Unfortunately the standard timer offered by the Linux kernel to meet these requirements not sufficient to fulfill these requirements.
This is because of the low timer resolution. So always a more precision timer is desirable.

Typically to get timer information in Linux, everyone uses  gettimeofday() system call, which returns clock time with microsecond precision.
To examine the timer precision, let's check with cyclictest to know how precise the calculation is with different timer.

By default, cyclictest uses POSIX timers. To see the default timer precision execute the below command.
```c
root@intel-corei7-64:~# cyclictest --loops=10000 --policy=fifo --priority=99
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.00 0.04 2.57 1/165 7462

T: 0 ( 7460) P:99 I:1000 C:  10000 Min:     14 Act:   43 Avg:   46 Max:     130
```
To see the timer precision with sys_settimer, execut the below

```c
root@intel-corei7-64:~# cyclictest --loops=10000 --policy=fifo --priority=99 --system
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.00 0.03 2.48 1/165 7474

T: 0 ( 7472) P:99 I:1000 C:  10000 Min:     15 Act:   40 Avg:   38 Max:     100
```
To see the timer precison with clock_nanosleep, execute below command.
```c
root@intel-corei7-64:~# cyclictest --loops=10000 --policy=fifo --priority=99 --nanosleep
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.00 0.04 2.53 1/165 7468

T: 0 ( 7466) P:99 I:1000 C:  10000 Min:      2 Act:   11 Avg:   10 Max:      19
```
Out of above all three test results, last timer provides the best performance. Where the measured latency is very minimal.
In order to analyze the results, refer below abbreviation and its description.

**T**   - Thread index(Thread ID)   
**P**   - RT Thread priority    
**I**   - Wake period of thread measuring latency (in microseconds)  
**C**   - Number of times the latency is measured   
**Min** - Minimum measured latency (in microseconds)    
**Act** - Actual measured latency (in microseconds)   
**Max** - Maximum measured latency(in microseconds)


Both of the above are based on signaling mechanism. Delivery of signals at the expiry of posix timers and itimer can not be done in the hard interrupt context of the high resolution timer interrupt. The signal delivery in both these cases must happen in thread context due to locking constraints that results in long latencies.

Below are the two APIs used by default POSIX timer and itimers respectively.

```c
//POSIX timers
int timer_create(clockid_t clockid,
                 struct sigevent *sevp,
                 timer_t *timerid);

//Itimers
int setitimer(int which,
              const struct itimerval *new_value,
              struct itimerval *old_value);
```

Where as clock_nanosleep does not work on signaling mechanism, which is why clock_nanosleep does not suffer above latency problem. Here the timer expiry is executed in the context of the high resolution timer interrupt.
So it is recommended if the an application is not using asynchronous signal handler better to use clock_nanosleep.

Clock_nanosleep use below API internally.
```c
//Clock_nanosleep
int clock_nanosleep(clockid_t clock_id,
                    int flags,
                    const struct timespec *request,
                    struct timespec *remain);
```

## High Resolution Timers on Linux

High resolution timers are being introduced in the Linux kernel, in order to offer a more precise way of waking up the system and process data at more accurate intervals. Which is especially interesting in the context of embedded systems.

Historically, x86 Linux systems uses timers with a frequency of 100Hz(i.e. 100 timer events per second/one event every 10ms).
With Linux kernel version 2.4, i386 systems started using timers with frequency of 1000Hz(i.e. 1000 timer events per second/one event every 1ms).
The 1ms timer event improves minimum latency and interactivity but at the same time it also incurs higher timer overhead. So, with Linux kernel version 2.6 timer frequency was reduced to 250HZ(i.e. 250 timer events per second/one event every 4ms) to reduce timer overhead.

In order to use high resolution timers, one needs to configure it in the Linux kernel config file as below:

```c
CONFIG_HIGH_RES_TIMERS=y
```
This will give the accuracy of sleep and timer system calls as accurate as the hardware(i.e ~microsecond resolution)

As an alternative, one can also examine the timer_list from /proc file system as below:

```
root@intel-corei7-64:~#cat /proc/timer_list | grep 'cpu:\|resolution\|hres_active\|clock\|event_handler'
```
```c
cpu: 0
 clock 0:
  .resolution: 1 nsecs
 #2: <ffffc9000214ba00>, hrtimer_wakeup, S:01, schedule_hrtimeout_range_clock.part.28, rpcbind/507
 #3: <ffffc900026d7d80>, hrtimer_wakeup, S:01, schedule_hrtimeout_range_clock.part.28, cleanupd/585
 #4: <ffffc9000269fd80>, hrtimer_wakeup, S:01, schedule_hrtimeout_range_clock.part.28, smbd-notifyd/584
 #8: <ffffc9000261fd80>, hrtimer_wakeup, S:01, schedule_hrtimeout_range_clock.part.28, smbd/583
 #9: <ffffc9000212bd80>, hrtimer_wakeup, S:01, schedule_hrtimeout_range_clock.part.28, syslog-ng/494
 #10: <ffffc900026dfd80>, hrtimer_wakeup, S:01, schedule_hrtimeout_range_clock.part.28, lpqd/587
 clock 1:
  .resolution: 1 nsecs
 clock 2:
  .resolution: 1 nsecs
 clock 3:
  .resolution: 1 nsecs
  .get_time:   ktime_get_clocktai
  .hres_active    : 1
cpu: 2
 clock 0:
  .resolution: 1 nsecs
 #2: <ffffc90002313a00>, hrtimer_wakeup, S:01, schedule_hrtimeout_range_clock.part.28, thermald/548
 #3: <ffffc900023fb8c0>, hrtimer_wakeup, S:01, schedule_hrtimeout_range_clock.part.28, wpa_supplicant/562
 clock 1:
  .resolution: 1 nsecs
 clock 2:
  .resolution: 1 nsecs
 clock 3:
  .resolution: 1 nsecs
  .get_time:   ktime_get_clocktai
  .hres_active    : 1
 event_handler:  tick_handle_oneshot_broadcast
 event_handler:  hrtimer_interrupt
 event_handler:  hrtimer_interrupt
```

- **.resolution** value of 1 nsecs, clock supports high resolution.   
NOTE: A resolution of 1ns is not resonable. This only tells, that the system uses HRTs. The usual resolution of HRTs on modern systems is in the micro seconds.
- **event_handler** is set to hrtimer_interrupt, this represents high resolution timer feature is active.
- **.hres_active** has a value of 1, this means high resolution timer feature is active.


In order to have the most definitive proof that your kernel supports high resolution timers, one can develop an isochronous application by their own.

## **Sample Isochronous Application**
An Isochronous application is one which will be repeated after a fixed period of time. The execution time of this application should always be less than its period. An isochronous application should always be a real time thread to measure performance.

**Steps to develop an isochronous application**		
Follow the below step by step procedure to develop a real time isochronous application:

**Application required header file**
```c
/*Header Files*/
#include <pthread.h>
#include <stdio.h>
#include <string.h>
```
**Define Data Structure**  
Define a structures that will have the time period information along with the current time of the clock. This structure will be used to pass the data between multiple tasks.
```c
/*Data format to be passed between tasks*/
struct time_period_info {
        struct timespec next_period;
        long period_ns;
};
```
**Periodic task Initialization**  
Define the time period of the periodic task to 1ms and get the current time of the system.

```c
/*Initialize the periodic task with 1ms time period*/
static void initialize_periodic_task(struct time_period_info *tinfo)
{
        /* keep time period for 1ms */
        tinfo->period_ns = 1000000;
        clock_gettime(CLOCK_MONOTONIC, &(tinfo->next_period));
}
```

**Timer increment**   
Timer increment module is used to go for nanosleep to complete the time period of the real_time_task.
```c
/*Increment the timer till the time eriod elapsed and again the Real time task will execute*/
static void inc_period(struct time_period_info *tinfo)
{
      tinfo->next_period.tv_nsec += tinfo->period_ns;
      while(tinfo->next_period.tv_nsec >= 1000000000){
        tinfo->next_period.tv_sec++;
        tinfo->next_period.tv_nsec -=1000000000;
      }
}

```
**Wait loop for time period completion**  
This loop will ne used for waiting for time period completion. Assumption here is task execution time time is less as compaed to time period
```c
/*Assumption: Real time task requires less time to complete task as compared to period length, so wait till period completes*/
static void wait_for_period_complete(struct period_info *pinfo)
{
        inc_period(pinfo);
        /* Ignore possibilities of signal wakes */
        clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &pinfo->next_period, NULL);
}
```
**Real-Time task**    
Define a Real-Time task here. For simplicity a print statement is kept here.
```c
static void real_time_task()
{
        printf("Real-Time Task executing\n");
        return NULL;
}
```
**Main thread for an isochronous application**   
Here the periodic task will be initialized and trigger the realtime task execution. Also will wait for the time period completion. This thread will be created from the main thread as a POSIX thread.

```c
void *realtime_isochronous_task(void *data)
{
        struct time_period_info tpinfo;
        periodic_task_init(&tpinfo);
        while (1) {
                real_time_task();
                wait_for_period_complete(&tpinfo);
        }
        return NULL;
}
```

**Non Real Time Master Thread**   
A non real time master thread will spawn a Real time isochronous application thread here. Also it sets the real time priority and scheduling class of the Real time task.

**Initialize the Thread attribute**   
A posix thread will be created here. So, before creating initialize the thread with the attributes.
```c
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
```
**Set Scheduling Policy**   
Next, Set the Real time thread with FIFO scheduling policy here.

```c
status = pthread_attr_setschedpolicy(&attr_fifo, SCHED_FIFO);
if (status) {
  printf("pthread_attr_setschedpolicy failed\n");
  return status;
}
```
**Set Task Priority**  
The Real time task priority is kept here as 92. The proority can be choose between 1-99.
```c
  	param_fifo.sched_priority = 92;
  	status = pthread_attr_setschedparam(&attr_fifo, &param_fifo);
  	if (status) {
  		printf("pthread_attr_setschedparam failed\n");
  		return status;
  	}
```
**Set the inherit-scheduler attribute**   
Set the inherit-scheduler attribute of the thread attribute. The inherit-scheduler attribute determines if new thread takes scheduling attributes from the calling thread or from the attr.
```c
  	status = pthread_attr_setinheritsched(&attr_fifo, PTHREAD_EXPLICIT_SCHED);
  	if (status) {
  		printf("pthread_attr_setinheritsched failed\n");
  		return status;
  	}
```    
**Spawn the Real Time Task**    
Real Time isochronous application thread will be created here.
```c    
  	status = pthread_create(&thread_fifo, &attr_fifo, realtime_isochronous_task, NULL);
  	if (status) {
  		printf("pthread_create failed\n");
  		return status;
  	}
```
**Wait for Real Time task completion**        
```c
  	pthread_join(thread_fifo, NULL);
  	return status;
  }
```

Find the complete code example below.

```c
/*Header Files*/
#include <pthread.h>
#include <stdio.h>
#include <string.h>

/*Data format to be passed between tasks*/
struct time_period_info {
	struct timespec next_period;
	long period_ns;
};

/*Initialize the periodic task with 1ms time period*/
static void initialize_periodic_task(struct time_period_info *tinfo){
	/*Keep time period for 1ms*/
	tinfo->period_ns = 1000000;
	clock_gettime(CLOCK_MONOTONIC, &(tinfo->next_period));
}

/*Increment the timer to till time period elapsed*/
static void inc_period(struct time_period_info *tinfo){
	tinfo->next_period.tv_nsec += tinfo->period_ns;
	while(tinfo->next_period.tv_nsec >= 1000000000){
		tinfo->next_period.tv_sec++;
		tinfo->next_period.tv_nsec -=1000000000;		
	}
}

/*Real time task requires less time to complete task as compared to period length, so wait till period completes*/
static void wait_for_period_complete(struct time_period_info *tinfo){
	inc_period(tinfo);
	clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &tinfo->next_period, NULL);
}

/*Real Time Task*/
static void* real_time_task(){
	printf("Real-Time Task executing\n");
	return NULL;
}

/*Main module for an isochronous application task with Real Time priority and scheduling call as SCHED_FIFO */
void *realtime_isochronous_task(void *data){

	struct time_period_info tinfo;
	initialize_periodic_task(&tinfo);

	while(1){
		real_time_task();
		wait_for_period_complete(&tinfo);
	}
	return NULL;
}

/*Non Real Time master thread that will spawn a Real Time isochronous application thread*/
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

  	status = pthread_create(&thread_fifo, &attr_fifo, realtime_isochronous_task, NULL);
  	if (status) {
  		printf("pthread_create failed\n");
  		return status;
  	}

  	pthread_join(thread_fifo, NULL);
  	return status;
}
```


## **NOHZ** (Tickless) Kernel

A CPU used to have multiple task simultaneously and every task should get fair share of CPU execution time. In order to shift CPU attention periodically towards multiple task, a scheduling clock interrupt is available with the Linux kernel to do this job. Traditionally, the Linux kernel used to send the scheduling clock interrupt (ticks) to each CPU every jiffy. Where jiffy is a very short period of time, which is determined by the value of the kernel Hz.

In case of input power constrained device like mobile device, triggering clock interrupt can drain its power source very quickly even if it is idle. In another situation like virtualization, multiple OS instance might find that half of its CPU time is consumed by unnecessary scheduling clock interrupts. In order to meet these kind of challenges, tickless linux kernel provides an option to reduce the number of scheduling clock interrupt, which helps to improve energy efficiency and reducing OS jitter. Reduction in OS jitter is very much important to Real-Time application as well as Computational intensive application.


Below are the three contexts where one needs to look for configuring scheduling-clock interrupts to improve energy efficiency:

**CPU with Heavy Workload**  
There are situations when CPU with heavy workloads with lots of tasks that use very short time of CPU and having very frequent idle periods, but these idle periods are also quite short (order of tens or hundreds of microseconds). In these cases, reducing scheduling-clock ticks will have reverse effect of increasing the overhead of switching to and from idle and transition between user and kernel execution. So better not to reduce the scheduling-clock ticks that might impact the performance.   

To achieve this mode of operation one, configure below in Kconfig file of Linux kernel using below:

```c
   CONFIG_HZ_PERIODIC=y
   CONFIG_NO_HZ=n        //for older kernels
```

**CPU in idle**  
The primary purpose of a scheduling-clock interrupt is to force a busy CPU to shift its attention among multiple tasks. But an idle CPU, no tasks to shifts its attention, therefore sending scheduling-clock interrupt is of no use. Instead, configure tickless kernel to avoid sending scheduling-clock interrupts to idle CPUs, thereby improving energy efficiency of the systems.

This mode can be achieved by enabling below configuration in Kconfig of Linux kernel.

```c
CONFIG_NO_HZ_IDLE=y
```
This mode can be disable from boot parameter by specifying "nohz=off". By default, kernel boot with "nohz=on".


**CPU with Single Task**   
In a case where CPU predefined with only one task to do, there is no point of sending scheduling-clock interrupt to switch task. So, in order to avoid sending-clock interrupt to this kind of CPUs, below configuration setting in Kconfig of Linux kernel will be useful.

```c
CONFIG_NO_HZ_FULL=y
```
 This will be very much useful for improving energy efficiency of the systems.

 Depending on the context one can select any of the mode from above three to improve energy efficiency and reducing OS jitter.
