# Memory Management
Memory management is considered as one of the important and critical aspects of Real Time Operating Systems as compared to a normal 
operating systems. There are different algorithms for Memory Management designed to optimize the runnable processes and improve system 
performance. If a process needs to be in the memory fully then kernel allocates the required memory to that process. 
For processes which needs only part of them to be in memory the memory management works along with scheduler for optimal utilization 
of resources.

We can take a look at three main areas of memory management - Memory Locking, Stack Memory and Dynamic Memory Allocation.
## Memory Locking
Memory locking is essential as part of the initialization the program. Most of the real-time processes lock the memory throughout their execution. Memory locking API locks the pages (virtual address space) of the application into the main memory. Locking of memory will make sure that the application pages are not removed from main memory during crisis. This will also ensures that page-fault do not occur during RT critical operations which is very important.

`mlock` and `mlockall` functions can be used by applications to lock the memory while `munlock` and `munlockall` are used to unlock the memory.<br/>
      `mlock(void *addr,size_t len)` - This function locks a selected region(starting from addr to len bytes) of address space of the calling process into memory.<br/>
      `mlockall(int flags)` - This function locks all of the process address space. MCL_CURRENT, MCL_FUTURE and MCL_ONFAULT are different flags available.<br/>
      `munlock(void *addr,size_t len)` - This function will unlock a specified region of a process address space.<br/>
      `munlockall(void)` - This system call will unlock all of the process address space.<br/>
      
## Stack Memory 
Each of the threads within an application have their own stack. The size of the stack can be specified by using the pthread function pthread_attr_setstacksize. If the size of the stack is not set explicitly then the default stack size is allocated. If the application uses a large number of RT threads, it is advised to use smaller stack size than default size. The default stack size on Linux is 2MB.<br/>
 Syntax of pthread_attr_setstacksize<br/>
	`pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize)`<br/>
	`attr` - thread attribute structure.<br/>
	`stacksize` - in bytes, should not be less than PTHREAD_STACK_MIN(16384 bytes).<br/>
  
## Dynamic Memory Allocation
Dynamic memory allocation of memory is not suggested for RT threads during the execution is in RT critical path as this increases the chance of page faults. It is suggested to allocate the required memory before the start of RT execution and lock the memory using the mlock/mlockall functions.

In the below example the thread function is trying to dynamically allocate memory to a thread local variable and try to access data stored in theses random locations.

```c
#define BUFFER_SIZE 1048576
void *thread_function_fifo(void *data) {
	double sum = 0.0;
	double* tempArray = (double*)calloc(BUFFER_SIZE, sizeof(double));
	size_t randomIndex;
	int i = 50000;
	while(i--)
	{ 
		randomIndex =  rand() % BUFFER_SIZE;
		sum += tempArray[randomIndex];
	}
 	return NULL;
  }
```

The above thread function is tested with and without `mlockall`. 

The below image shows the `perf report` output when tested without using `mlockall` 

![2](https://github.intel.com/storage/user/9783/files/835b9312-a005-11e8-863b-d7584ee29233)

The below image shows the `perf report` output when tested using `mlockall` 

![1](https://github.intel.com/storage/user/9783/files/4984d734-a005-11e8-8e8a-eb8c9a35eae1)


From the above observations we can see that the number of page faults in the thread function is more when locking is not used.
