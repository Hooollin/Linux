## The Process
*process* is a program in the midst of execution. Typically they are made of: *text section*, resources such as open files and pending signals, internal kernel data, processor state, a memory address space with one or more memory mappings, one or more threads of execution and a *data section* containing global variables.  

Linux has a unique implementation of threads: It does not differentiate between threads and processes. To Linux, a thread is just a special kind of process.  

In Linux, a process begins its life when at the system call of **fork()**. The process that calls **fork()** is the *parent)*, and the new process is the *child*. They execute at the same place: where the call to **fork()** returns.  

Notice that **fork()** is actually implemented by via the **clone()** system call.  

**exit()** terminates the process and frees all its resources. A parent process can inquire about the status of a terminated child via the **wait()** system call, which enables a process to wait for the termination of a specific process. When a process exits, it is placed into a special zombie state that represents terminated processes unti the parent calls **wait()** or **waitpid()**.  

## Process Descriptor and the Task Structure
The kernel stores the list of processes in a circular doubly linked list called the *task list*. Each element in the task list is a *process descritor* of the type **struct task_struct**, which is defined in **<linux/sched.h>**.  

The **task_struct** is relatively large data structure, at around 1.7 kilobytes on a 32-bit machine. The process descriptor contains the data that describes the executing program - open files, the process's address space, pending signals, the process's state, and much more.  

### Allocating the Process Descriptor
The **task_struct** structure is allocated via the *slab allocator* to privde object reuse and cache coloring(?). **struct task_struct** was stored at the end of the kernel stack of each process, which allowed architectures with few registers to caculate the location of the process descriptor via the *stack pointer* without using an extra register to store the location. Because the process descriptor now dynamically created via the slab allocator, a new structure, **struct thread_info**, was created that again lives at the bottom of the stack and at the top of the stack.

![The process descriptor and kernel satck](static/Figure%203.2.png)

### Storeing the Process Descriptor
The system identifies proceses by a unique *process identification* value or PID. The kernel stores this value as **pid** inside each process descriptor.  

**pid** is a type of **pid_t**, which is typically an **int**. Because of backward compatibility with earlier Unix and Linux version, the default maximum value is only 32,768 (that of a **short int**).  

This maximum value is important because it is essentially the maximum number of processes that may exist concurrently on the system. Although 32,768 might be sufficient for a desktop system, lare servers may require many more processes. The lower the value, the sonner the values will wrap around, destroying the useful notion that higher vlaues indicate later-run processes than lower values.  

We can configure and to break compatibility with old applications, set pid to a much higher value.  

Tasks are typically referenced directly by a pointer to their **task_struct** structure inside the kernel. It is useful to be able to quickly look up the process descriptor of the currently executing task ,which is done via the **current** macro. However, this macro must be independently implemented by each architecture. Some architectures svae a pointer to the **task_struct** structure of the currently running process in a register, enabling for efficient acess. Other architectures, make use of the fact that **strcut thread_info** is stored on the kernel stack to calculate the location of **thread_info** and subsequently the **task_struct**.

### Process State
The **state** field of the process descriptor describes the current condition of the process. Each process on the system is in exactly one of five different states:
- TASK_RUNNING - The process is runnable;it is either currently running or on a run-queue waiting to run. This is the only possible state for a proces executing in user-space;it can also apply to a process in kernel-space that is actively running.
- TASK_INTERRUPTIBLE - The process is sleeping (or to say it is blocked), waiting for some condition to exist.
- TASK_UNINTERRUPTIBLE - This state is identical to TASK_INTERRUPTIBLE except that it does not wake up and become runnable if it receives a signal. This is used in situations where the process must wait without interruption or when the event is expected to occur quite quickly. (?)
- __TASK_TRACED - The process is being *traced* by another process, such as a debugger, via ptrace.
- __TASK_STOPPED - Process execution has stopped; the task is not running nor is it eligible to run. This occurs if the task receives the **SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU** or if it receives *any* signal while it is being debugged.
![Flow chart of process states](static/Figure%203.3.png)

### Manipulating the Current Process State
**set_task_state(task, state);**

### Process Context
System calls and exeptions handlers are well-defined interfaces into the kernel. A process can begin executing in kernel-space only through one of these interfaces - *all* access to the kernel is through these intefaces.

### The Process Family Tre
A distinct hierarchy exists between processes in Unix systems.   

All processes are descendants of the **init** process, whose PID is one. The kernel starts **init** in the last step of the boot process. The **init** process, in turn, reads the system initscripts and executes more programs, eventually completing the boot process.  
Every process on the system has exactely one parent. Every process has zero or more child.  

Each **task_struct** has a pointer to the parent's **task_struct**, named **parent**, and a list of children, named **children**.  It's easy to obtain the parent process or iterate over a process's children.  

The **init** task's process descriptor is statically allocated as **init_task**.  

## Process Creation
Process creation in Unix is unique. Most operating system implement a *spawn* mechanism to create a new process in a new address space, read in an executable, and begin executing it. Unix taks the unusual approach of separating these steps into two distinct functions: **fork()** and **exec()**.  

**fork()** creates a child process that is a copy of the current task. It differs from the parent only in its PID, its PPID, and certain resources and statistics, such as pending signals, which are not inherited. 

**exec()** loads a new executable into the address space and beings executing it.  

### Copy-on-Write
On **fork()**, all resources owned by the parent are duplicated and the copy is given to the child. This approach is naive and inefficient in that it copies much data that might otherwise be shared. If the new process were to immediately execute a new image, all that copying would go to waste.  

In Linux, **fork()** is implemented through the use of *copy-on-write* pages. COW is a technique to delay or altogether prevent copying of the data. Rather than duplicate the process address space, the parent and the child can share a single copy.  

The data, is marked in such a way that if it is written to, a duplicate is made and each process receives a unique copy. Thus, the duplication of resources occurs only whne they ar written; until the, they are shared read-only. This technique delays the copying of each page in the address space unti it is actually written to.  If those shared pages are never written to, no copies are needed.  

### Forking

Linux implements **fork()** vis **clone()** system call, and **clone()** in turn, calls **do_fork()**.

**do_fork()** calls **copy_rpocess()** and then starts the process running. 

Steps of **copy_process()**:
1. It calls **dup_task_struct()**, which creates a new kernel stack, **thread_info** structure, and **task_struct** for the new process. The new values are identical to those of the current task at the point.
2. It checks that the new child will not exceed the resource limit on the number of processes for the current user.
3. Make child different from its parent.
4. Set child's state to TASK_UNINTERRUPTIBLE to ensure that it does not yet run.
5. Update the flags member of the **task_struct**.
6. **alloc_pid()** for the new task.
7. Depending on the flags passed to clone(), some of resources are duplicated.
8. Return a pointer to the child process.

Linux will deliberately run the child process first to eliminates any copy-on-write overhead that would occur if the parent ran first and began wiring to the address space. 

### vfork()
Before COW technique was introduced to Linux, vfork() is a optimization of fork that it does not copy the parent's page table entries. 

## The Linux Implementation of Threads
Threads are a popular mordern programming abstraction. They provide multiple threads of execution within the same program in a shared memory address space. They can also share open files and other resources. Threads enable *concurrent programming* and even true *parallelism* on multiple processor systems.

Linux has a unique implementation of threas. To the Linux kernel, there is no concept of a thread. Linux implements all threads as standard processes. 

A thread is merely a process that shares certain resources with other processes. Each thread has a unique **task_struct** and appears to the kernel as a normal process - threads just happen to share resources, such as an address space, with other processes.


T.his approach to threas contrasts greatly with operating systems such as Windows or Solaris, which has explicit kernel support for threads (or *lightweight processes*)

### Creating Threads
Threads are created the same as normal tsks, with the exception that the **clone()** system call is passed flags corresponding to the specific resources to be shared:
``clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0);``
The code results in behavior identical to a normal **fork()** except that address space, filesystem resources, file descriptors and signal handlers are shared. This new task and its parent are what are popularly called *threads*.

### Kernel Threads
It is often useful for the kernel to perform some operations in the background. (why to have kernel threads)

*kernel threads* - standard processes that exist solely in kernel-space. The significant difference between kernel threads and normal processes is that kernel threads do not have an address space. (They're created in kernel space)

Linux delegates several tasks to kernel threads, most notably the *flush* task and the *ksoftirqd* task. 

## Process Termination
When a process terminates, the kernel releases the resources owned by the process and notifies the child's parent of its demise.

Generally, process destruction is self-induced. It occurs when the process calls the **exit()** system call, either explicitly when it is ready to terminate or implicitly on return from the main subroutine of any program.  

A process can also terminate involuntarily. This occurs when the process receives a signal or exception it cannot handle or ignore. 

No matter how a process terminates, the builk of the work is handled by **do_exit()**,, which completes a number of chores:
1. It sets the **PF_EXITING** in the **flags** member of the **task_struct**.
2. It calls **del_timer_sync()** to remove any kernel timers. Upon return, it is guaranteed that no timer is queued and that no timer handler is running.
3. Maybe do some accounting information work.
4. **exit_mm()** will be called to release the **mm_struct** held by this process. If no other process is using this address space, the kernel then destroys it.
5. It calls **exit_sem()** to dequeued from an IPC semaphore queue.
6. It calls **exit_files()** and **exit_fs()** to decrement the usage count of objects related to file descriptors and filesystem data, respectively.
7. It sets the task's exit code, stored in the **exit_code** member of the task_struct, to the code provided by **exit()** or whatever kernel machanism forced the termination.
8. It calls **exit_notify()** to send signals to the task's parent, reparents any of the task's children to another thread in their thread group or the init process, and sets the task's exit state, stored in **exit_state** in the **task_struct** structure, to **EXIT_ZOMBIE**.
9. **do_exit()** calls **schedule()** to switch to a new process. This process is now not schedulable, this is the last code the task will ever execute. It will never return.

At this point, all objects associated with the task are freed. It's a so called ZOMBIE. The only memory it occupies is its kernel stack, the **thread_info** structure, and the **task_struct** structure. It's exists solely to provide information to its parent. After the parent retrieves the information (**wait()** or wait family system calls), or notifies the kernel that it is uninterested, the remaining memory held by the process is freed and returned to the system for use. 

### Removing the Process Descriptor
