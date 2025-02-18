# Lab 2 - Scheduler

## Accept Your Assignment

[https://classroom.github.com/a/sFukyoo3](https://classroom.github.com/a/sFukyoo3)

**Due: Sunday, Feb 23rd, 2025, 23:59, PST**

## Background

If you have been familiar with preemptive/non-preemptive scheduling, FIFO, RR, Priority-Based scheduling algorithms. You can skip to [ticks in xv6](#ticks-in-xv6-riscv)

### What is scheduler

The scheduler is a critical component of an operating system kernel responsible for managing and prioritizing access to the CPU(s) among multiple processes or threads. Its primary role is to decide which task runs next and for how long, ensuring efficient resource utilization and fair allocation.

### Preemptive v.s. Non-Preemptive

A scheduling strategy is non-preemptive if the scheduling happens only when a running task voluntarily releases the CPU, typically when it:

* Completes execution.
* Blocks for I/O or synchronization (e.g., `sleep()`, `wait()`)

Modern OS like Linux/Windows/macOS all use preemptive scheduling. 

For preemptive scheduling, the OS kernel forcibly interrupts a running task to allocate CPU time to another task. A preemptive scheduling is often triggered by:

* Time-slice exhaustion (e.g., Round Robin scheduling)
* Arrival of a higher-priority task.

A preemptive scheduling typically relies on hardware timer. 

In xv6-riscv, the initial scheduler is a preemptive round robin scheduler. 

### Scheduling Algorithms

#### First-In, First-Out (FIFO)

FIFO (also called First-Come, First-Served, FCFS) is a non-preemptive CPU scheduling algorithm where processes are executed in the exact order they arrive in the ready queue.

Key Principles:

* Non-Preemptive Execution
    * A process retains the CPU until it completes or voluntarily blocks (e.g., for I/O).
    * No interruption occurs mid-execution, even for higher-priority processes.
* Queue Structure
    * Processes are managed in a FIFO queue.
    * Newly arriving processes join the end of the queue.
    * Pick the first process in the queue for running.

----------------

#### Round Robin (RR)

Round Robin is a preemptive algorithm. It ensures fair CPU time allocation by cycling through processes in a fixed order. 

**Key Principles:**

* Time Slice (Quantum)
    * Each process is assigned a fixed time quantum (e.g., 10-100ms in real world, 1 CPU cycle in xv6-riscv). 
    * The process runs until it either:
        * Completes within its time slice.
    	* Is interrupted when the time slice expires. 
* Ready Queue
    * Processes are arranged in a First-In-First-Out (FIFO) queue.
    * After a process uses its time slice, it is moved to the end of the queue, and the next process runs. 

**Example Workflow:**

Assume we have three processes (P1, P2, P3) with a time quantum of 4ms:

| Time (ms) | Action                        | Ready Queue State |
| --------- | ----------------------------- | ----------------- |
| 0         | P1 runs (4ms allocated)       | [P2, P3]          |
| 4         | P1 preempted -> P2 runs       | [P3, P1]          |
| 8         | P2 preempted -> P3 runs       | [P1, P2]          |
| 12        | P3 preempted -> P1 runs again | [P2, P3]          |

**Advantages:**

* Fairness: No process starves; all get equal CPU time over cycles. 
* Responsiveness: Ideal for interactive systems (e.g., GUIs) where tasks need frequent updates. 
* Simplicity: Easy to implement and predict.

**Disadvantage:**

* Overhead: Frequent context switches waste CPU time if the quantum is too small.
* Throughput: Less efficient than priority-based schedulers for long-running tasks. 

----------------

#### Priority-Based

Priority-Based Scheduling allocates CPU time to processes based on their assigned priority levels. Processes with higher priority are executed first, making it ideal for systems where urgent tasks must preempt less critical ones (e.g., real-time systems).

**Key Principles:**

* Priority Assignment
    * Each process is assigned a priority value (numeric, e.g., 0-99, where 0 = highest priority.
    * Priorities can be:
    	* Static: Fixed at process creation (e.g., real-time tasks).
    	* Dynamic: Adjusted during runtime (e.g., aging to prevent starvation).
* Preemption Modes
    * Preemptive: A higher-priority process immediately interrupts a running lower-priority one.
    * Non-Preemptive: A running process completes its burst time before yielding to higher-priority tasks.
    * *Attention:* the preemption modes here is different from preemptive/non-preemptive algorithm. Priority-Based scheduling algorithm is a preemptive scheduling algorithm, while it has two preemption modes (preemptive/non-preemptive when a higher-priority process occurs). 

### Burst Time, Turnaround Time and Waiting Time

**Burst Time**

We use burst time representing the continuous execution time of a single CPU occupation by a process. 

For example, `p` is scheduled to run by scheduler at time $t_0$ and it finishes/interrupted by kernel at time $t$. The burst time for this execution of `p` is $t-t_0$. 

We also use total burst time representing the total CPU occupation time by a process. 

For example, `p` was scheduled for `4` times and the burst times are $[t_0, t_1, t_2, t_3]$. The total burst time of `p` for now is $t_0 + t_1 + t_2 + t_3$. 

**Turnaround Time**

Assume process `p` was created at time $t_0$ and `p` finished at time $t$. Turnaround time of finished process `p` is $t-t_0$. It's the total time for finishing a process. 

**Waiting Time**

Assume process `p` was created at time $t_0$ and current time is $t$. For now, `p`'s total burst time is $t_b$. The waiting time of `p` for now is $(t-t_0) - t_b$. $t-t_0$ is the total time from `p` is created to now. Waiting time is the total time subtracted total burst time. 

## Ticks in xv6-riscv

In real world OS, we evaluate the time using cycles or seconds. In xv6-riscv, there is another unit for evaluating the time, which is `ticks`. 

`ticks` is defined at the beginning of `trap.c`. It's defined globally, therefore its initial value is 0. `ticks` will be incremented whenever a time quantum is used. 

The execution flow path for incrementing `ticks` is `usertrap()` -> `devintr()` -> `clockintr()`. 

You can get current `ticks` by using system call `uptime()` in user space program. If you are in kernel space, you can directly access variable `ticks`, which is defined in `def.h` as an external variable. Its real declaration is in `trap.c`. 

We will use `ticks` as the unit for evaluating the time in this lab. 

## Scheduler in xv6-riscv

The initial scheduler in xv6-riscv is a RR scheduler. Before we get into the scheduler itself, we need understand how this RR scheduler works in xv6-riscv. 

------------

**Periodic Interrupts for OS (in Supervision mode)**

One important but often neglected part in time-sliced RR scheduler is the periodic interrupts in the kernel. RR scheduler requires periodic interrupts from the hardware so that it can preempt the execution of current process. 

In xv6-riscv, related code is in `/kernel/start.c`. There is a function named `timerinit()`, which initializes the timer. 

```c
// ask each hart to generate timer interrupts.
void
timerinit()
{
  // enable supervisor-mode timer interrupts.
  w_mie(r_mie() | MIE_STIE);
  
  // enable the sstc extension (i.e. stimecmp).
  w_menvcfg(r_menvcfg() | (1L << 63)); 
  
  // allow supervisor to use stimecmp and time.
  w_mcounteren(r_mcounteren() | 2);
  
  // ask for the very first timer interrupt.
  w_stimecmp(r_time() + 1000000);
}
```

------------

**Interruption Handler**

We have met this handler before in system call. System call is a kind of interrupt from user space to kernel space and this timer interrupt is from user space to kernel space as well. So they are both trapped by the same function `usertrap()` in `trap.c`. 

```c
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call
	...
    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
	...
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

At the end of this function, there is a if statement `which_dev == 2` checking whether this interrupt is triggered by timer. We can see `yield()` function is called if it's a timer interrupt. 

------------

**Yield Function**

The `yield()` function is in `proc.c`. 

```c
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

It gets the current running process `p` and sets its state with `RUNNABLE`, which means `p` has finished his burst time. 

`sched()` function is called afterwards. This function does a context switch from process `p` and current kernel process.

Now we need to know, before `p` gets to run, the control is in `scheduler()` function. Line 31 is the point where user space process `p` is executed and also it's the point to be returned whenever `p` finishes its running. 

```c
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for(;;){
    // The most recent process to run may have had interrupts
    // turned off; enable them to avoid a deadlock if all
    // processes are waiting.
    intr_on();

    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);
		// (1)
        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
        found = 1;
      }
      release(&p->lock);
    }
    if(found == 0) {
      // nothing to run; stop running on this core until an interrupt.
      intr_on();
      asm volatile("wfi");
    }
  }
}
```

1. User space process is switched out from here. After the process finishes, `sched()` function will switch back to this point of scheduler function and start the next round of scheduling. 

> Context switch function is written in assembly code and it's in `swtch.S`. 

------------

**Scheduler**

This function is not hard to understand. It consists of a big infinite `for` loop. In each round, it picks up a `RUNNABLE` process and run it if possible (for loop from line 22-38). If it cannot find a `RUNNABLE` process (if statement at line 39), it will enable interrupt and use `wfi` instruction telling CPU to stop running until the next interrupt. 

For process picking, this function goes over the process list, find a `RUNNABLE` process and run it. That's it. This is just a simply RR scheduling implementation. 

## Locks in `struct proc`

**What are Locks**

In computer science, locks are synchronization mechanisms used to control access to shared resources in a multi-threaded or multi-process environment. They ensure that only one thread or process can access a critical section of code or a shared resource at a time, preventing race conditions and maintaining data consistency.

**Why are Locks Needed**

In modern operating systems and applications, multiple threads or processes often run concurrently and may need to access shared resources (e.g., variables, data structures, files). Without proper synchronization, concurrent access can lead to:

Race Conditions: When the behavior of a program depends on the timing of thread execution, leading to unpredictable or incorrect results.
Data Corruption: When multiple threads modify shared data simultaneously, resulting in inconsistent or corrupted data.
Locks solve these problems by enforcing mutual exclusion: only one thread can hold a lock at a time, ensuring that shared resources are accessed in a controlled and safe manner.

**How do Locks Work**

A lock typically has two main operations:

Acquire (or Lock): A thread requests ownership of the lock. If the lock is available, the thread acquires it and proceeds to the critical section. If the lock is held by another thread, the requesting thread waits (blocks) until the lock is released.

Release (or Unlock): The thread that currently holds the lock releases it, allowing other threads to acquire the lock and access the critical section.

**Types of Locks**

* Mutex (Mutual Exclusion Lock):
	* A basic lock that allows only one thread to access a resource at a time.
	* Example: Protecting a shared variable or data structure.
* Spinlock:
    * A lock where the thread repeatedly checks (spins) until the lock becomes available.
    * Used in scenarios where waiting times are expected to be short.
* Semaphores:
    * A generalized lock that allows a specified number of threads to access a resource simultaneously.
    * Can be used for more complex synchronization scenarios.

------------

**Locks in `struct proc` of xv6-riscv**

In fact, we have used `locks` in lab 1. In `wait()` system call, there are multiple function calls like `acquire(...)` and `release(...)` and if you didn't handle these function calls properly, there might be a panic during execution. 

Here, we'd like to learn these locks and know when do we need to use `locks`. 

In `proc.h`, the definition of `struct proc`, there are three types of fields and they are well classified into three categories-`p->lock` must be held, `wait_lock must be held` and `private`. 

In `wait()`, because multiple CPUs could access `state`, `xstate` and `pid` simultaneously, `p->locks` is needed when accessing these fields. 

In this lab, you may need to add several fields like `priority`, `start_time` and `burst_time` in `struct proc`. These fields should belong to the first category-`p->lock` required. **Please include the reason for this in your report.**

## Your Tasks

### Overview

Your task for this Lab is implementing a time-sliced priority-based scheduler with aging strategy. 

------------

*Time-sliced priority-based scheduler*

A time-sliced priority-based scheduler is quite similar to a RR scheduler. It also runs each process with a fixed time slice. The only difference is that, RR scheduler picks the process one by one while our scheduler will always pick the process with the highest priority (**to make it simple, if multiple processes have the same priority, pick the most previous one in the process list**).

------------

*Aging*

Aging strategy is used for solving starvation in Priority-Based scheduling. In Priority-Based scheduling, if a process with higher priority requires a lot of time to finish, it will take the CPU for a long time, which causes other processes hardly running. 

To solve this, instead of having a fixed priority value for each process, the scheduler adjusts the priority value for processes during dynamically. A simple case is that, every time the scheduler picks a process `p` to run, `p`'s priority will be deceased and other processes' priority value increase.

You will be required to apply a simple aging strategy to avoid starvation in this lab. 

------------

$3$ more system calls are required to implement for this lab.

### Breakdown

**Time-Sliced Priority-Based Scheduler**

* A time-sliced priority-based scheduling. 
	* Based on the RR scheduler of xv6-riscv.
	* Add a field in `struct proc` representing the scheduling priority of this process. We use an `unsigned char` or `uint8` type for this priority. It's value ranged from $0$ to $255$. **$0$ means the highest priority and $255$ is for the lowest priority.** The default priority for a process should be $128$
	* When picking the next process for execution, instead of picking the very first `RUNNABLE` one in the queue, pick the very first one with the highest priority in the queue. 
* Aging strategy.
  * When a process is executed by the scheduler, its priority value will be increased by $1$. 

**Extra System Calls**

> All definition requirements are for the user space. 

<!-- -----------
```c
// Returns the total burst time (in ticks) of the process with `pid`. 
int ticks(int pid);
```

Example: Process `p` was scheduled for $3$ times by scheduler and it finished all its time slice. When I call `ticks(p)`, it should return $3$. 

Return the ticks for current process if `pid` isn't valid (i.e., there is no process with `pid`). 

--------------
```c
// Get the waiting time for process with `pid`
int waiting_ticks(int pid);
```

Example: Process `p` is created at tick $10$. Current time is tick $20$. `p` was scheduled for $3$ times during this time. The waiting time is $20-10-3=7$. 

Return the waiting time (in ticks) for current process if `pid` is not valid (same as above). -->

-------------
```c
// Set the priority value of current process with p. 
int setpriority(uint8 priority);
// Get the priority value of process with `pid`. If `pid` isn't valid, return the one of current process.
int getpriority(int pid);
```

`setpriority(0)` will set the priority value of current process (which calls `setpriority()` function) to `0`. 
`getpriority(5)` will return the priority value of process with `pid` == 5 if there is. Otherwise, return the priority value of current process.

-------------

We have learned this in lab 1. The original definition of `wait()` is `int wait(int *status)`. In this lab, we also need to implement another version of `wait()`, which is `wait_and_get_ticks()`.

The definition of `wait_and_get_ticks()` is 
```c
int wait_and_get_ticks(int* status, int* ticks, int* turnaround_ticks);
```

Besides the status, which is the return value of the child process needs to be stored and passed to parent process, another two fields `ticks`-the execution time (in ticks) and `turnaround_ticks`-the turnaround time (in ticks) need to be stored to the the pointer `ticks` and `turnaround_ticks` as well. 

> `ticks` and `turnaround_ticks` are in the user space. Handle this carefully. 

## Step-by-Step Instruction

There is no step-by-step instruction for lab 2.

## Implementation Tips

1. Implement the three system calls first. It is well-introduced in lab 1 about how to make new system calls.
2. The major modification lays on `scheduler()` function in `proc.c`. Current scheduler uses RR algorithm and you can start from it for our time-sliced priority-based scheduler. 
3. You may need to add additional fields in `struct proc` or other structure  to assist `scheduler()` and/or extra system calls. 
4. You may need to set `CPUs` variable in `Makefile` to $1$ to see a sequential & explainable results. 

## Grading Policy

10 pts in total:

1. 2 pts for the attendance. 
2. 6 pts for the Lab code (2 pts will be shown in autograder, 4 pts will be graded manually). If you don't have a report submitted, your grade for the code will receive a 50% penalty. 
3. 2 pts for the Lab report.

We reserve the rights to grade your Lab according to your report in any situation. You can make a request for grading your Lab just according to your report if you cannot submit a runnable code. But we can't ensure we will give you any grade. 

We may also grade your assignment just according to your report (which means we will give up your grade in autograder) if we find any **SENSITIVE** code in your work or we find some unmatched performance for you between the labs and lectures. To avoid this case, understand and following the instruction well, write the code by yourself and never share your code with others. 

## What to write in report

We don't expect a very detailed report. Keep it in 2-3 pages. You don't need to copy, paste and explain your code in the report, as we can find them in your repo. 

Instead, you need to explain

1. Which files you modified and the reason for the modification. 
2. Is there any difficulty you met and how do you solve them.
3. Why is `p->lock` required when accessing the fields like `priority`, `burst time` etc. in the scheduler?
4. How do you deal with the `locks` in your scheduler. (*1 report pts*)
5. Where do you add the logic for aging and the reason for this. (*1 report pts*)
6. Are there any disadvantages for our time-sliced priority-based scheduler. If so, what are they and are there any methods to improve them? Otherwise explain why you think it's perfect. 

You can add other things in the report as well if you like. 

## Test Program

```c
#include "kernel/types.h"
#include "user/user.h"

char *token = "Q3Jvc3NpbmcgdGhlIHJpdmVyIGJ5IGZlZWxpbmcgdGhlIHN0b25lcw==";

unsigned int seed = 1;

void srand(unsigned int x) {
    seed = x;
}

unsigned int lcg_rand(void) {
    seed = (1103515245 * seed + 107) % (1 << 8);
    return seed;
}

int abs(int x) {
    return x < 0 ? -x : x;
}

int test_priority_setter_getter() {
    printf("test_priority_setter_getter====================\n");
    printf("[test_priority_setter_getter] setting self's priority to 0\n");
    setpriority(0);
    if (getpriority(0) != 0) {
        printf("[test_priority_setter_getter] setpriority(0) failed, expected getpriority(0) == 0, got %d\n", getpriority(0));
        return 1;
    } else {
        printf("[test_priority_setter_getter] getpriority(0)=%d\n", getpriority(0));
    }
    int priority = lcg_rand() % (1<<8);
    printf("[test_priority_setter_getter] setting self's priority to %d\n", priority);
    setpriority(priority);
    if (getpriority(getpid()) != priority) {
        printf("[test_priority_setter_getter] setpriority(%d) failed, expected getpriority(%d) == %d, got %d\n", priority, priority, getpid(), getpriority(getpid()));
        return 1;
    } else {
        printf("[test_priority_setter_getter] getpriority(%d)=%d\n", getpid(), getpriority(0));
    }
    printf("[test_priority_setter_getter] %s\n", token);
    return 0;
}

int test_wait_and_get_ticks() {
    printf("test_wait_and_get_ticks====================\n");
    int sleep_time = lcg_rand() % 5 + 5;

    int pid = fork();
    if (pid == 0) {
        printf("[test_wait_and_get_ticks] child process, pid: %d, sleep_time: %d\n", getpid(), sleep_time);
        sleep(sleep_time);
        exit(0);
    }

    int status, ticks, turnaround_ticks;
    int actual_pid = wait_and_get_ticks(&status, &ticks, &turnaround_ticks);
    
    if (pid != actual_pid) {
        printf("[test_wait_and_get_ticks] expected pid %d, got %d\n", pid, actual_pid);
        return 1;
    } else {
        printf("[test_wait_and_get_ticks] waited pid: %d\n", actual_pid);
    }

    if (status != 0) {
        printf("[test_wait_and_get_ticks] expected status 0, got %d\n", status);
        return 1;
    } else {
        printf("[test_wait_and_get_ticks] status: %d\n", status);
    }
    printf("[test_wait_and_get_ticks] ticks: %d\n", ticks);

    printf("[test_wait_and_get_ticks] turnaround_ticks: %d\n", turnaround_ticks);

    printf("[test_wait_and_get_ticks] %s\n", token);
    return 0;
}

int test_scheduler() {
    printf("test_scheduler====================\n");
    
    int childs[4];
    int prioritys[] = {128, 130, 135, 145};
    int final_priority[4];
    setpriority(0);

    for (int i = 0; i < 4; ++i) {
        childs[i] = fork();
        if (childs[i] == 0) {
            printf("[test_scheduler] child process, pid: %d, priority: %d\n", getpid(), prioritys[i]);
            setpriority(prioritys[i]);
            while(1);
            exit(0);
        }
    }

    sleep(170);
    int x = 0;
    for (int i = 0; i < 4; ++i) {
        final_priority[i] = getpriority(childs[i]);
        x += final_priority[i] - prioritys[i];
        printf("[test_scheduler] child process, pid: %d, final priority: %d\n", childs[i], final_priority[i]);
        if (final_priority[i] != 177) {
            printf("[test_scheduler] expected priority 177, got %d\n", final_priority[i]);
            // return 1;
        }
        kill(childs[i]);
    }
    printf("[test_scheduler] total priority difference: %d\n", x);
    printf("[test_scheduler] %s\n", token);
    return 0;
}

int (*tests[])(void) = {
    test_priority_setter_getter,
    test_wait_and_get_ticks,
    test_scheduler
};

int all_tests() {
    for (int i = 0; i < sizeof(tests) / sizeof(tests[0]); ++i)
        if (tests[i]()) return 1;
    return 0;
}

int main(int argc, char **argv) {
    srand(uptime());
    if (argc > 1) return tests[atoi(argv[1])]();
    else return all_tests();
}
```

For test 3-`test_scheduler`, it doesn't mean your scheduler is correct when you pass this test. There is a desirable output for it, which is the final priority for each process is `177` and the total difference priority difference is `170`. However, these numbers are not absolute. You answer should be close to this answer but minor difference is also acceptable. 