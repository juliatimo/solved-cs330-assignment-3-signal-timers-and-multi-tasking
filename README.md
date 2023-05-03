Download Link: https://assignmentchef.com/product/solved-cs330-assignment-3-signal-timers-and-multi-tasking
<br>
<strong>Recap: </strong>gemOS is in 64-bit mode executing itself as the first context (say the boot context). The boot context sets up the page table, stack, segment registers for itself. Further, it implements basic input output to a serial console, and puts itself into a basic shell. At this stage, GemOS implements a command called init which creates the first user process (named as the init process with PID = 1). Source code for init process can be found in user/init.c file. The current init process supports five system calls—getpid(), exit(), write(), expand() and shrink(). gemOS also implements lazy memory allocation by handling page fault exception. Further, divide-by-zero exception is also handled by the gemOS. Now we are ready to take the next step.

Objectives of the assignment are to implement new system calls (signal(), alarm(), sleep() and clone()) along with basic signal handling and multitasking using a round robin (RR) scheduler. To enable multi-tasking design of gemOS, a periodic timer interrupt is initialized with a handler function handle timer tick defined in schedule.c. Every invocation of the periodic timer interrupt handler is counted as one <em>tick</em>. Note that, the timer interrupt handler has the same semantic of a div-by-zero fault handler, albeit with a separate interrupt stack.

For this assignment, a list of contexts is maintained in gemOS—accessed using get ctx list(), which returns the pointer to the first process (PID = 0). You can iterate the list as an array of pointers using PID as an index. Currently, the maximum number of contexts is defined by a macro MAX PROCESSES which is 16 (PID=0,1…15). For more details, please refer to the definitions of struct exec context and process states in include/context.h and include/schedule.h, respectively. A template of the required implementation is provided in schedule.c file.

<h2>Part A – Signals</h2>

In this part of the assignment, you are required to implement the signal handling functionality for three signals (enumerated in include/schedule.h file) and described as follows,

<ul>

 <li><em>SIGSEGV: </em>This signal corresponds to an invalid access of a memory location by the program.</li>

 <li><em>SIGFPE: </em>This signal corresponds to a divide-by-zero operation by the user program.</li>

 <li><em>SIGALRM: </em>This is an alarm signal generated after every <em>numticks </em>number of timer interrupts where <em>numticks </em>is specified using alarm() system call. For example, alarm(5) will set the <em>numticks </em>to 5. See the man page—man alarm for more details.</li>

</ul>

For all of the above signals, signal handlers are registered using signal(signo, handler) system call that is required to be implemented as part of the assignment. In the extended definition of struct exec context, a bit vector and a signal handler array is provided to maintain the pending signals and the handlers, respectively. In the design of the signal handling mechanisms, assume that there will be no nested signal handler invocations. For SIGSEGV and SIGFPE, the function invoke sync signal is invoked from the exception handlers which has the following semantics,

long invoke_sync_signal(int signo, u64 *ustackp, u64 *urip)

<em>signo </em>is the signal number. <em>ustackp </em>is the pointer to the stack pointer location in the exception entry stack. <em>urip </em>is the pointer to the instruction pointer location in the exception entry stack.

The alarm(ticks) system call should initiate counting of the ticks using ticks to alarm member of struct exec context while maintaining the original ticks in alarm config time member. When the ticks expire, a signal must be sent to the user space if the signal handler for SIGALRM is registered, ignored otherwise. Note that, you are required to invoke the invoke sync signal function with appropriate interrupt stack pointers to deliver this signal.

Note that, for this part of the assignment, uni-process test cases (only with init process) will be used and not be tested with features mentioned below.

<h2>Part B – System calls (sleep()) and Swapper process</h2>

As part of this assignment, you are required to implement sleep functionality for the init process.

int sleep(u32 ticks)

Suspends the execution of the calling context for <em>ticks </em>number of timer ticks. During this time, the context is moved to WAITING state (see struct exec context in include/context.h) and the swapper process (with pid=0, already created in the system which is in ready state) is loaded. If timer interrupts arrive while the swapper process is RUNNING, the ticks should be accounted for (using ticks to sleep member of struct exec context) and depending on the remaining ticks either the swapper process is rescheduled or the sleeping context is scheduled.

Initially, the swapper process context along with the regs member which is of type struct user regs is initialized in a manner such that if the currently used OS stack (the current context OS stack or the interrupt stack) is loaded with the last five elements before executing iretq, the swapper process is scheduled. In general, this strategy may be employed to switch between any two processes.

Please note that, before performing the actual switch to the new process (see schedule context() in the template code), your code must invoke set tss stack ptr(next) and set current ctx(next), where next is the incoming context.

<h2>Part C – clone() and scheduling</h2>

In this part of the assignment, you are required to implement context creation functionality using clone() system call. Further, you are required to implement a round robin scheduler to schedule the contexts (in READY state) in the system.

int clone(void *th func, void *user stack) <em>th func </em>is the pointer to the function that will be executed by the newly created context.

<em>user stack </em>is the pointer to the stack which will be used by the newly created context. This has to be a memory location in the MM SEG DATA region after expansion (using expand() system call followed by initialization) .

Assume that the above two virtual memory parameters are always correct and not required to be validated by walking the page tables of the calling process. The system call handler for clone() must create a new context which is a copy of the parent process with the below mentioned exceptions. You should use get new ctx() declared in include/context.h to allocate a new context. The pid of the returned context will be already initialized and the status will be set to NEW. All other fields of the context should either be initialized or copied from the parent context. The values which are not copied are,

<ul>

 <li>os stack pfn must be allocated for the new context using os pfn alloc(OS PT REG)</li>

 <li>name must be the name of the parent appended with the pid value of the context. You may use memcpy() call declared in include/lib.h.</li>

 <li>regs must be appropriately initialized so that when it is scheduled, the new context executes the th func using user stack. You may set the values of SS and CS to 0x2b and 0x23, respectively (see slide-20 of Userland.pdf). The value of RFLAGS must be set to the RFLAGS value of the parent.</li>

</ul>

Please note that, this clone() implementation is neither a thread or a process. This is because, even though the CR3 is the same, the mm segments field is copied and separate for the parent and the newly created context. Therefore, to avoid cases with memory issues, clone() will be invoked from the main process (i.e., init) only. After the creation of the new context, the parent returns from system call and the child state is set to be READY to be scheduled. Note that, the behavior of exit() system call should be modified (please see the template for do exit() in schedule.c). The exit() system call should free the os stack pfn and change the process state to UNUSED. Further, if there are no other contexts in the system except the swapper process, it must invoke cleanup(). Otherwise, the scheduler should be invoked to schedule the next READY process or the swapper process.

The scheduler can be invoked in three different ways—when a process exits, when a process blocks using sleep() system call or on a timer interrupt. In the RR scheduling scheme, the next READY process in the list after the currently running process (in a circular order including the current process as the end) is scheduled after every tick. If there are no ready contexts in the system, swapper process must be loaded. Note that, whenever a context switch happens, the PID of old and new must be printed.