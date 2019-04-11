## Introduction to PREEMPT_RT

Linux is a full-featured operating system, originally designed to be used in server or desktop environments. Since then, Linux has evolved and grown to be used in almost all computer areas i.e. embedded systems, cloud servers. Due to its wide popularity, excellent performance and a good protocol stack implementation Linux is the de facto choice for many real-time systems. Linux has the similar design of the UNIX OSs with features like stability, robustness and a secure system.

A Real-Time Operating System (RTOS) should be able to serve an application in real-time i.e. it should not add any delays to process an application request. Current Linux mainline is not completely real-time capable. However, with PREEMPT_RT patch Linux kernel gets the real-time OS capabilities.

#### Precedent Work

Here we will talk about the work that lead to the development of the PREEMPT_RT patch and also different ways which allows to enable real-time functionality in the Linux ecosystem.

##### Early Linux Kernels

In early Linux kernel (Linux 1.x), tasks could not be preempted once they started executing code in the kernel or in a driver. Preemption could occur only when the task exited the kernel and scheduling took place after the task voluntarily suspended itself in the kernel.

The Linux 2.x kernel series introduced SMP (Symmetric multiprocessing) support. Multiple threads could execute in the kernel as long as they did not attempt to access the same shared data at the same time. The tasks that arrived later had to wait, if multiple tasks competed for a same critical section. In the 2.4.x versions of Linux, the maximum latency is very high.
The 2.6 kernel series added features and improvements to mainstream which evolved from the need of the embedded systems. This version included a new scheduler feature. This version also had a feature to interrupt a task executing the kernel code (only few parts of the kernel code is pre-emptible).
The PREEMPT_RT patch which is based on kernel preemption added support for full critical section preemption and interrupt handlers to Linux kernel 2.6.

In 3.4 version of the kernel new scheduling policy Earliest Deadline First(EDF) is added.


#### Classification of Linux-based RTOSs:

There are various approaches to add real-time features to Linux kernel. These approaches can be grouped into following
1. Interrupt Abstraction  
2. Kernel Preemption approaches

Now, we will dig deeper into these two approaches to provide the real-time capabilities in Linux.


#### 1. Interrupt Abstraction  

In this approach a virtual hardware layer is created between standard Linux kernel and computer hardware. This layer is called as Real-Time Hardware Abstraction Layer. A complete real-time subsystem runs in parallel with Linux OS, this subsystem consists of a RTOS and required device drivers. The interrupts in this system are labeled as real-time or non-real-time. The real-time interrupts are handled by real-time subsystem and non-real-time interrupts are handled by Linux kernel. RTLinux, RTAI and Xenomai are some of implementations of Interrupt Abstraction approach.


##### Real-Time Application Interface (RTAI)

RTAI is a real-time extension to Linux, the implementation is based on RTLinux. An Adeos (Adaptive Domain Environment for Operating Systems) based patch introduces Real-Time Hardware Abstraction Layer (RTHAL) to RTAI. This layer intercepts and process the hardware interrupts. RTAI has its own scheduler. The RTAI provides features like IPC, Memory Management and POSIX threads.

##### Xenomai

Xenomai provides a set of RTOS services that can be patched to Linux kernel. Xenomai provides interrupt virtualization by using Adeos nanokernel. It supports features like multithreading, timers, memory allocation and synchronization support.  Xenomai allows running of RT applications in user space or kernel space. Xenomai adopts the concept of multiple entities which can be called as domains. These domains exist on the same machine simultaneously. Xenomai is parted into primary domain and secondary domain. A primary domain runs a real-time component that is named as real-time nucleus and a secondary domain runs Linux OS.

##### Advantages

Interrupt Abstraction is effective in reduction of latency. The real-time tasks are executed in kernel space and hence the response times is very less. This enables faster interrupt response and task switching. Interrupt virtualization allows running non-real-time tasks on a Linux OS and real-time tasks on the real-time OS. This helps in efficiently running the critical tasks as time critical tasks are run by RTOS and non-real-time critical tasks can be run on Linux OS.

##### Limitations

As the execution of the real-time tasks happens in the kernel space, both kernel and RT task share same memory space and execute with same privilege. Because of which there is no protection to the memory between them.

Real-time tasks are loaded as dynamic modules, due to which if there is any error in the real-time task it may crash the whole system.

Finally, it requires specific drivers to be developed for the real-time system, plus patches to Linux itself in order to work besides it. As the communities are not as big as Linux', this may pose a challenge.

#### 2.Kernel Preemption approaches

In 2.4 version of Linux kernel two approaches are proposed for reducing the kernel latency - low latency patch and pre-emptible kernel patch.

* In the low latency patch explicit rescheduling point called as preemption points are inserted in the code that may take long intervals of time for execution.
* In the preemption approach a task with higher priority can preempt a lower priority task even if it is running in kernel mode. This is the feature required for real-time systems.

##### The PREEMPT_RT patchset

`PREEMPT_RT` patch makes it possible for Linux pre-emption to enjoy real-time characteristics, by introducing e.g. thread interrupts, sleeping spinlocks or priority inheritance.

It pursues bringing down the number of lines of non-pre-emptible kernel code. This is achieved by adding and modifying functionality in the Linux kernel. Some of the changes done by ```PREEMPT_RT``` patch are:
* Running interrupt handlers as kernel threads. This enables preemption of IRQ handler by higher priority thread while servicing an interrupt.
* Converting spin locks to mutexes. This enables preemption while holding a lock.
* Adding priority inheritance to threads of lower priority. This avoids scenarios where a higher priority thread waits for resource acquired by lower priority thread which is preempted by a thread that has intermediate priority. This adds more time for the lower and higher priority threads to get executed.
* High resolution timers

###### Integration of PREEMPT_RT with Linux

After the advent of PREEMPT_RT patch it is accepted widely and has active developer community. The main aim of the PREEMPT_RT patch is to bring real-time features into the mainline Linux kernel. Most of the real-time features of the PREEMPT_RT are added in the mainline kernel versions of 2.6.x.

There are several releases of the PREEMPT_RT patch over the time, the latest version is 4.9-rt. The other versions of the PREEMPT_RT which are actively maintained include versions 4.18-rt, 4.14-rt, 4.9-rt, 4.4-rt, 3.18-rt, and 3.10-rt.

The latest mainline Linux kernel have most of the PREEMPT_RT features integrated. A detailed list of these features include: high resolution timers, threaded interrupt handlers, mutex infrastructure, pre-emptible and hierarchal read-copy-update, generic timekeeping, priority inheritance, generic interrupt handling or tracing improvements. However, for full realtime preemption support, some patching is still necessary. The [PREEMPT_RT patches are available in the Linux Foundation website](https://wiki.linuxfoundation.org/realtime/preempt_rt_versions). The efforts are on by developers to merge the PREEMPT_RT patch into the mainline kernel.

Now, in the next labs we will deep dive into real-time Scheduling with PREEMPT-RT, Affinity Tuning, Power Management and other important topics
