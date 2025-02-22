# Lab 1 - System Call

!!! warning
    **Jan 15th, 9:30am**: I opened a new pull request on some of your repos. If you accept the assignment before **Jan 15th, 9:30am**, please kindly refer to [here](./pull-request.md) about how to merge it. 

!!! bug
    **Jan 17th, 9:50am:** I updated the test code to fix a bug. This bug didn't affect the result on GitHub Classroom. It only affected the result when you test it on your local machine. 

!!! bug
    **Jan 21th, 3:55pm:** I updated several invisible file for debugging. If you want to debug using gdb, it would show you something like `no target .gdbinit.tmpl-riscv can be compiled`. This PR fixed this bug. 

## Accept Your Assignment

[https://classroom.github.com/a/c8VO7Hjn](https://classroom.github.com/a/c8VO7Hjn)

**Due: Sunday, Feb 2nd, 2025, 23:59, PST**

## Background

### What is system call?

A system call is a way for programs to interact with the operating system. It provides an interface between a process and the operating system, allowing the process to request high-privilege services such as file operatios, process control and communication. 

Because it involves two parties - the user program (low privilege) and the operating system (high privilege), it is bound to have a transition of privileges. 

### How does system call work?

You can call a system call (e.g., exit(), which is used for terminating current program) in a user space program (like in our hello-world.c program). Let's just use our hello-world program and change it a little bit.

```c
// hello-world.c in /user/
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char **argv) {
	printf("Hello, world!\n");
	exit(0);
	printf("?.? Dead code ?.?\n");
}
```

Also, we need to add a line in the `Makefile`. This modification adds a user space program, making program `make` know that it should compile the program `hello-world.c` under `/user/`. `$U` stands for `/user/` directory. 

> What is user space program and what is kernel space program? If you take a look at the codebase of xv6, you may find there are two major directories, `/user/` and `/kernel/`. All the things in `/kernel` directory is kernel space program/code and the so for `/user`. However, there are much more different things between them. You can learn more from [wiki](https://en.wikipedia.org/wiki/User_space_and_kernel_space#:~:text=Kernel%20space%20is%20strictly%20reserved,one%20address%20space%20per%20process.)

```makefile
... line 138
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_hello-world\
```

Let's compile this file and see what will happen. 

```bash
$ make qemu
riscv64-linux-gnu-gcc    -c -o kernel/entry.o kernel/entry.S
...
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ 
```

Then use `ls` command listing all available programs in user space. 

```bash
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2292
cat            2 3 34256
echo           2 4 33184
forktest       2 5 16168
grep           2 6 37520
init           2 7 33640
kill           2 8 33096
ln             2 9 32912
ls             2 10 36280
mkdir          2 11 33152
rm             2 12 33144
sh             2 13 54720
stressfs       2 14 34040
usertests      2 15 179344
grind          2 16 49392
wc             2 17 35216
zombie         2 18 32520
hello-world    2 19 32656
console        3 20 0
```

We can check there is a hello-world program. Then we can execute it by:
```bash
$ hello-world
Hello, world!
```

What happened? The code after `exit(0)` is just ignored. The reason for this is the program just exits at `exit(0)` and of course the left code wouldn't be executed forever. 

Ok, now let's just see what will happend in the call of `exit(0)`. 

The definition of `exit()` function is in the file `/user/user.h`.

```c
...
// system calls
int fork(void);
int exit(int) __attribute__((noreturn));
int wait(int*);
...
```

We can see that its return value is an integer and it receives an inteter as its parameter. The `__attribute__((noreturn))` part is a compiler annotation telling the compilers (gcc & clang both support this) this function wouldn't return. 

However, this is just the definition of this function. In C/C++, we know that a definition of function isn't executable. Therefore, where is the implementation of this function?

The secret is in the file `/user/usys.pl`. This file is a Perl program. Perl is just like Python, which is also an interpreter-based language. 

This `usys.pl` would generate the 'trigger' for function calls. This Perl script might be dense, but once you compile xv6 using `make qemu`, you can find the file generated by it, which is `/user/usys.S`. 

```asm
# generated by usys.pl - do not edit
#include "kernel/syscall.h"
...
.global exit
exit:
 li a7, SYS_exit
 ecall
 ret
...
```

Generally, this `/user/usys.S`, which is an assembly code for RISC-V, has done two things-including `/kernel/syscall.h` and making many global symbols (declared by `.global **`, e.g., `.global exit` defines a symbol `exit`) with corresponding functions' assembly code.

Let's just use the example of `exit()`.

```asm
.global exit        # declare a symbol, you can image it as a variable
exit:               # this is a tag, meaning symbol exit points to here
 li a7, SYS_exit    # load immediate value, SYS_exit, into register a7
 ecall              # system call instruction in RISC-V
 ret                # return
```

There are still several questions in this *small* piece of assembly code.

1. Where is the immediate value `SYS_exit`?
2. Where does the program go after we use `ecall` function?

For the first question, let's get back to the first thing `usys.S` has done-the included file `/kernel/syscall.h`

```c
// System call numbers
#define SYS_fork    1
#define SYS_exit    2
#define SYS_wait    3
...
```

See? All the immediate values are define here and included into the assembly code by `#include "kernel/syscall.h"`. 

For the second question, it's a little complicated. If you want to learn more, you can refer to [xv6 book](https://pdos.csail.mit.edu/6.1810/2024/xv6/book-riscv-rev4.pdf) chapter 4.3. In short, `ecall` instruction in user mode (now we are running `hello-world`, which is a userspace program) will trap the execution into `uservec()` function in `trampoline.S` and further get into `usertrap()` function in `/kernel/trap.c`. 

So far, the transition between user space and kernel space has done. The CPU's privilege mode has been set to 1, which is supervisor mode (S-mode). However, the memory page table hasn't been set to kernel's (you can think about the reason after you learn more). 

```c
void
usertrap(void)
{
...
	intr_on();

	syscall();
} else if((which_dev = devintr()) != 0){
...
}
```

Following function `usertrap()` in `/kernel/trap.c`, it finally goes to `syscall(void)` function in `/kernel/syscall.c`. This function is easy to understand now (T.T compared to previous assembly code). 

```c
void
syscall(void)
{
	int num;
	struct proc *p = myproc();

	num = p->trapframe->a7;
	if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
		// Use num to lookup the system call function for num, call it,
		// and store its return value in p->trapframe->a0
		p->trapframe->a0 = syscalls[num]();
	} else {
		printf("%d %s: unknown sys call %d\n",
						p->pid, p->name, num);
		p->trapframe->a0 = -1;
	}
}
```

At line 137, we got the number stored in user program's register `a7`. Feel familiar? Yes! It was stored a little before in the function `exit()` of `/user/usys.S`.

```asm
.global exit        # declare a symbol, you can image it as a variable
exit:               # this is a tag, meaning symbol exit points to here
 li a7, SYS_exit    # load immediate value, SYS_exit, into register a7 HERE!!!!!!!!!
 ecall              # system call instruction in RISC-V
 ret                # return
```

`SYS_exit`, which is just the SYSTEM CALL NUMBER, was stored in user program and then read by kernel here. Magic, right?

Then there is a `if` statement judging whetehr the syscall number is greater than 0 and less than the number of elements in `syscalls` table. OK, now let's take a look at `syscalls` table. (It's from line 107 - 129)

```c
// An array mapping syscall numbers from syscall.h
// to the function that handles the system call.
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
...
};
```

Here, this type might be hard to understand. To be simple, it's an array and each element in this array is a [*function pointer*](https://www.geeksforgeeks.org/function-pointer-in-c/). The functions should have type of `uint64 function(void)`, which means these functions receive no arguments and return a single unsigned 64-bits integer. 

Then, what's the values in this array? Let's use a simple C program example.

```c
#include <stdio.h>

int main() {
	int arr[] = {
		[0] 10,
		[1] 100,
		[3] 1000,
		[5] 0xdeadbeef
	};

	int len = sizeof(arr) / sizeof(int);
	printf("Length of arr: %d\n", len);
	for (int i = 0; i < len; ++i) {
		printf("arr[%d] = %d\n", i, arr[i]);
	}
}
```

You can simply copy this code and run it on your computer. The result should be:
```bash
$ ./main
Length of arr: 6
arr[0] = 10
arr[1] = 100
arr[2] = 0
arr[3] = 1000
arr[4] = 0
arr[5] = -559038737
```

Now, it's pretty obvious! The numbers in `[]` is the index and the number after each index is it's corresponding value. For example, we stored 10 to position 0 and 1000 to position 3. 

> Note: this semantic isn't officially supported by C++, which means even it's a valid grammer in C, it might be failed in compilation for some C++ compilers. 

Now, let's come back to the `syscalls` array. There are some functions and each function is stored at the position of its system call number. For example, the syscall number for `exit()` is 2, which you can find in `/kernel/syscall.h`. Therefore `syscalls[2]` will be a function pointer pointing to `sys_exit()`. 

We are close!

Then, what is `sys_exit()`? It's in file `/kernel/sysproc.c` and you can find it's pretty short. But before we get into the implementation of the function, we should know how can an array in `/kernel/syscall.c` use a function defined in `/kernel/sysproc.c`. If we want to use functions from another file, we typically define it in a header file and implement it in a source file. Whenever we want to use it, include the header file and call it. During the compilcation, we just need to pass these two source files to the compiler, it will find the function and link them. But... the situation here is a little different. As there is no header file defining the `sys_**` functions, functions in `/kernel/syscall.c` cannot find them. So, if you pay attention to the code just before the `syscalls[]` table, you will see there are a lot of `sys_**` functions defined as extern function. The key word `extern` tells the compiler, "this function's implementation is not here, but it will be somewhere else. I will give the source file including this function to you later. Now, you just assume you have already had the implemetation of this function. " 

Good? To learn more about `extern` key word, you can refer this [stack overflow thread](https://stackoverflow.com/questions/856636/effects-of-the-extern-keyword-on-c-functions). 

Now let's get into the implementation of `sys_exit()`.

```c
uint64
sys_exit(void)
{
	int n;
	argint(0, &n);
	exit(n);
	return 0;  // not reached
}
```

Do you still remember what the definition of `exit()` syscall is in the user space? It's `int exit(int) __attribute__((noreturn))`. It has one argument and one return value, both are of integer type. But now, when coming to `sys_exit()`, the return value is still there, but where is our argument?

Here, we need to explain why we cannot directly pass our parameters using our normal way (i.e., `sys_exit(int)`). It is because when we stores the function pointers to `syscalls[]`, all functions should have the same type. However, different system call has their own type of parameters. Hence, for a more convenient call of system calls of different arguments type, we make all of them with non-argument and let them parse their own arguments in `sys_**()`. (No matter you understand or not, I'm confused now :( )

In the first line of the function, it prepare a value `n` for the argument passed for `exit()` syscall. `argint(0, &n)` fetches the first value in the calling argument and stores it into variable `n`. Then it execute `exit(n)`. (Can it be the last function containing `exit`????)

OK, now let's see the very LAST `exit()` function. It's in the file `/kernel/proc.c`. However, before we step into the actual function implementation, we need know how `sys_exit()` finds `exit()` first. These two functions are defined in two differen  files, in order for one to find another, there should be a "bridge" for them. As we mentioned before, the most common bridge is a header file. Here, the same, there is a header file storing all definition of kernel functions (just like `/user/user.h`), which is `/kernel/defs.h`. You can also find it by doing a small search about the function definitions. 

> Searching, finding where the definition and implementation is a very important skill in C/C++ programming. The documentation is sometimes (tbh, always) unreliable. Therefore, most of time you need to learn from the code by yourself. Currently, the C/C++ extension in VSCode is very powerful and you can tack the function call flow by simply using `ctrl + left click` (or `command + left click` on macOS). However, sometimes it may skip some definition. Therefore, search and filter by yourself is most reliable way. 

```c
// Exit the current process.  Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void
exit(int status)
{
	struct proc *p = myproc();

	if(p == initproc)
		panic("init exiting");

	// Close all open files.
	for(int fd = 0; fd < NOFILE; fd++){
		if(p->ofile[fd]){
			struct file *f = p->ofile[fd];
			fileclose(f);
			p->ofile[fd] = 0;
		}
	}

	begin_op();
	iput(p->cwd);
	end_op();
	p->cwd = 0;

	acquire(&wait_lock);

	// Give any children to init.
	reparent(p);

	// Parent might be sleeping in wait().
	wakeup(p->parent);
	
	acquire(&p->lock);

	p->xstate = status;
	p->state = ZOMBIE;

	release(&wait_lock);

	// Jump into the scheduler, never to return.
	sched();
	panic("zombie exit");
}
```

This function is well documented so I wouldn't write a lot to explain it. But we can see, in the end, it calls `sched()` which is the process scheduler. When it gets into the scheduler, the other ready processes will be run by the scheduler. 

So, question is, where is our argument, the status for exit? It is stored in `p->xstate`, which is used for storing the exit status specially. 

> I wrote all of this by my own. Hopefully it can help you understand how function call works. T_T

## Your Task

There is only one task for you, easy, not much work to do and only a few dozen lines of code could finish it. 

Now, you may (or may not) know that, `fork()` will create a child process. Now, suppose you have one parent process with PID=1 and two child processes of him, PID=2 and PID=3. What if your parent want to wait for these two child processes finishing their work?

You can use `wait()` syscall. `wait()` will return once there is one child process exits (in another word, becomes *zombie* as we saw in `exit()`) after cleaning up all the memory for the child process. 

Easy to understand, right?

Your task, is adding another system call with user space definition

```c
int waitpid(int *status, int pid, int options);
```

Explanation for parameters:

1. `int *status`, this is a pointer, the status of exited program should be stored into this location, just like `wait(int*)`. 
2. `int pid`, instead of waiting for arbitrary child process terminating, `waitpid()` should only wait for specific child with `pid`. To be more specific:
	1. If pid <= 0, which means it's an invalid pid in xv6 (xv6 has a pid > 0), your function should work as `wait(int*)`, which means it will return whenever a child process exits. 
	2. If pid > 0, but the process with `pid` isn't the child of current process, your function should return -1. 
	3. If pid > 0 and the process with `pid` is the child of current process, the child process is running at the same time, your function should just do `wait` for this process with specified `pid`. 
3. `options`, we'd play something in the function for this options. The `options` in Linux kernel is complex and it defines many macro for different function (you can take a look from [here](https://linux.die.net/man/2/waitpid)). We don't need that complex `options`. Instead, we'd just use several values to change the operation of the function.
	1. If `options == 0` just run as normal (do nothing extra).
	2. If `options == 1`, use `printf()` (you may be curious about this `printf()`, this `printf()` is different from the one we used in normal program, this one is defined under the kernel space of xv6, even though they have the same function.) to output a line of `OPTIONS=1` and a new line character `\n`. Then your function should do nothing and return 0.
	3. If `options == 2`, do as normal, but when you store the status code of the child process, don't use the exact one, use a fake one `0x33` and the other things should be the same. 
	4. If `options == 3`, do nothing and return immediately with return value -1. 

## Step-by-Step Instruction

> Attention: as C program compilation is very flexible, it's not necessary for you to following the instruction so strictly. You can design, create, define and implement the function as you wish as long as it can work properly. If you aren't familiar to C programming or you haven't understand the background very well, you can start Lab 1 by just following the instructions here. 

So, following our mind in the background, let's start from user space. 

-----------------

The first thing is adding a definition for our syscall in user space. It's inside the file `/user/user.h`. 

> Don't understand the different between header file (.h file) and source file (.c)? Take a look at [here](http://www.math.uaa.alaska.edu/~afkjm/csce211/handouts/SeparateCompilation.pdf) (from UAA). 

Take a look at our required user space definition for the system call and finish this part. 

-----------------

The second thing is adding the syscall's implementation under user space. Now, take a little review in background, where is the syscalls' implementation in user space?

Remember that Perl script and generated assembly code? Yes, it's in `/user/usys.S`. However, this file is not written directly by us, instead it's generated by the Perl script. Don't worry if you don't understand the Perl script. Take a look at the script file `/user/usys.pl` and it's not hard to understand, right? The script sets a subroutine `entry` and it will print the corresponding entry assembly code for each syscalls defined here. Therefore, in order to add our `waitpid()` syscall, we need to add an entry here as well. 

-----------------

OK, now you may have noticed that, in `/user/usys.pl`, it includes a file `/kernel/syscall.h` and it uses variable `SYS_${name}` defined in it. We have seen this file before and there are several system calls and its syscall number defined. So, pick up a desirable system call number and define it in `/kernel/syscall.h`. 

> There are 21 system calls defined in xv6 and they have consecutive syscall numbers. It's natural for us picking a number 22 for our `waitpid()` syscall. But, is it really necessary? Can I choose other numbers here and what's the influence if I choose a very large number? If you cannot come up with an answer, remind yourself a little about the `syscalls[]` array we described in the background. 

-----------------

As you may notice, we are in the kernel level now. So, what is next step? It looks like all the clues stop here. 

It's the time to review how a system call is triggered by the kernel-the trap, right? There is a trap file and it calls something related to `syscall()` (if you don't remember, just go back to the background). 

Let go to `usertrap()` function and it seems there is nothing we can do here and it will finally get into function `syscall()` in `/kernel/syscall.c`. Here, we notice a old friend, the `syscalls[]` table, right? We have defined our syscall numbers in `/kernel/syscall.h`, therefore we have already had a constant number variable with name `SYS_**` corresponding to our syscall and here, we need to associate the syscall number with a concrete system call function, just like the association between the number `SYS_exit` (i.e., this number is 2 as we find in `syscall.h`) and `sys_exit()` function. 

-----------------

OK, we make a connection between them, but wait, where is our system call function? 

Firstly, there are some `extern` function defined just before this `syscalls[]` table, in order for our table can find the function, we should define an extern function corresponding to our `waitpid()` syscall here as well. 

Then, let's find where `sys_exit()` is and it's in the file `/kernel/sysproc.c`. Do you remember, here, the `sys_exit()` prepare the arguments for the real `exit()` function. Similarly, we need to define a function here preparing arguments for the real `waitpid()` function. 

Emmm... but the question is, we can learn from `sys_exit()` that if we want to fetch a integer from the argument, we need to use `argint()`. We have `pid` and `option` in integer, but how can I fetch a type `int*` for the argument? Don't worry, do you still remember `wait(int* status)` function call? There is a pointer as well! So, take a look at that function and you can find what you need. 

Then, just finish this part by yourself. 

-----------------

Now, we come to the most challenge part in the Lab and you need to implement the most of your code here, by yourself. We can follow the `sys_exit()` and it will call `exit(int)` in `/kernel/proc.c`. Similarly, you can define your `waitpid()` function here. 

But stop for a minute. Do you still remember, in order to let `sys_exit()` find `exit()`, there is a definition for `exit()` in `/kernel/defs.h` and it is included in `/kernel/sysproc.c`. Therefore, make a definition for our `waitpid()` in `/kernel/defs.h` so that our `sys_waitpid()` could find the definition for this function. 

Then, what to do? The most intuitive thing is that you can copy the code from `wait(int*)` function as they perform two similar things. Now, it's time to take a look at [your tasks](#your-task) and implement it according to the requirements. 

> If you meet any confusion during finishing the Lab, don't hesitate to ask! It might be my bad. Maybe I made an unreasonable requirement or I miss something in the code. 

## Grading Policy

10 pts in total:

1. 2 pts for the attendance. 
2. 8 pts for the Lab code (as shown in autograder if you submit your report). If you don't have a report submitted, your grade for the code will receive a 50% penalty. 

We reserve the rights to grade your Lab according to your report in any situation. You can make a request for grading your Lab just according to your report if you cannot submit a runnable code. But we can't ensure we will give you any grade. 

We may also grade your assignment just according to your report (which means we will give up your grade in autograder) if we find any **SENSITIVE** code in your work or we find some unmatch performance for you between the labs and lectures. To avoid this case, understand and following the instruction well, write the code by yourself and never share your code with others. 

> The grades on Github Actions might not be correct if you partially passed test cases. The grades on the table is always calculated by $\frac{8.0}{5} \times \text{PASSED CASES}$. But the final grades below is always correct. Please refer to `Test` section above `autograding reporter` for the actual grades distribution.


## What to write in report

We don't expect a very detailed report. Keep it in 2-3 pages. You don't need to copy, paste and explain your code in the report, as we can find them in your repo. 

Instead, you need to explain

1. Which files you modified and the reason for the modification. 
2. Is there any difficulty you met and how do you solve them. 
3. If let you pick some special functions for more options, what will you choose and do you have some reality reason for this choice. (e.g., there are many options for `waitpid()` in the real world Linux and they all have their corresponding existing reasons). 

You can add other things in the report as well if you like. 

## Test Program

```c
#include "kernel/types.h"
#include "user/user.h"

/******************************/
// (1)
unsigned int seed = 1;

void srand(unsigned int x) {
    seed = x;
}

unsigned int lcg_rand(void) {
    seed = (1103515245 * seed + 107) % (1 << 8);
    return seed;
}
/******************************/
// (2)
int test_waitpid(void) {
    int nproc = 10;
    int status[nproc];
    int childs[nproc];
    // initialize return status for child processes
    for (int i = 0; i < nproc; ++i) status[i] = lcg_rand();

    for (int i = 0; i < nproc; ++i) {
        int pid = fork();
        childs[i] = pid;
        if (pid == 0) {
            sleep(lcg_rand()%5 + 1);
            exit(status[i]);
        }
    }
    printf("[test_waitpid] created %d processes\n", nproc);
    for (int i = nproc-1; i >= 0; --i) {
        int actual_status;
        int pid = waitpid(&actual_status, childs[i], 0);
        printf("[test_waitpid] waiting for pid[0x%x] with expected xstate[0x%x]\n", pid, status[i]);
        if (pid != childs[i]) {
            printf("[test_waitpid] waitpid: expected pid[0x%x] but got pid[0x%x]\n", childs[i], pid);
            return 1;
        }
        if (actual_status != status[i]) {
            printf("[test_waitpid] waitpid: expected xstate[0x%x] for pid[0x%x] but got [0x%x]\n", status[i], pid, actual_status);
            return 1;
        }
    }
    printf("[test_waitpid] Sinocentric Epoch\n");
    return 0;
}
/******************************/
// (3)
int test_waitpid_pid_le_0() {
    int nproc = 10;
    int status[nproc];
    int childs[nproc];
    // initialize return status for child processes
    for (int i = 0; i < nproc; ++i) status[i] = lcg_rand();

    for (int i = 0; i < nproc; ++i) {
        int pid = fork();
        childs[i] = pid;
        if (pid == 0) {
            sleep(lcg_rand()%5 + 1);
            exit(status[i]);
        }
    }
    printf("[test_waitpid_pid_le_0] created %d processes\n", nproc);
    for (int i = nproc-1; i >= 0; --i) {
        int actual_status;
        int pid = waitpid(&actual_status, -1, 0);
        printf("[test_waitpid_pid_le_0] get pid[0x%x] return with xstate[0x%x]\n", pid, actual_status);
        int flg = 0;
        for (int j = 0; j < nproc; ++j) {
            if (pid == childs[j]) {
                childs[j] = -1;
                flg = 1;
                if (actual_status != status[j]) {
                    printf("[test_waitpid_pid_le_0] waitpid: expected xstate[0x%x] for pid[0x%x] but got [0x%x]\n", status[j], pid, actual_status);
                    return 1;
                }
                break;
            }
        }
        if (!flg) {
            printf("[test_waitpid_pid_le_0] pid[0x%x] not a valid child\n", pid);
            return 1;
        }
    }
    printf("[test_waitpid_pid_le_0] Sinocentric Epoch\n");
    return 0;
}
/******************************/
// (4)
int test_waitpid_pid_not_child() {
    int pid = fork();
    if (pid == 0) {
        sleep(lcg_rand()%5 + 1);
        exit(0);
    }
    int wait_pid = pid - 1;
    int status;
    int ret = waitpid(&status, wait_pid, 0);
    if (ret != -1) {
        printf("[test_waitpid_pid_not_child] waitpid: expected -1 but got retval[0x%x]\n", ret);
        return 1;
    }
    printf("[test_waitpid_pid_le_0] Sinocentric Epoch\n");
    return 0;
}
/******************************/
// (5)
int test_waitpid_option_1() {
    int pid = fork();
    if (pid == 0) {
        sleep(lcg_rand()%5 + 1);
        exit(lcg_rand());
    }
    int status;
    int ret = waitpid(&status, pid, 1);
    if (ret != 0) {
        printf("[test_waitpid_option_1] waitpid: expected 0 but got retval[0x%x]\n", ret);
        return 1;
    }
    ret = waitpid(&status, pid, 0);
    if (ret != pid) {
        printf("[test_waitpid_option_1] waitpid: expected pid[0x%x] but got pid[0x%x]\n", pid, ret);
        return 1;
    }
    printf("[test_waitpid_option_1] Sinocentric Epoch\n");
    return 0;
}
/******************************/
// (6)
int test_waitpid_option_2() {
    int pid = fork();
    if (pid == 0) {
        sleep(lcg_rand()%5 + 1);
        exit(lcg_rand());
    }
    int status;
    int ret = waitpid(&status, pid, 2);
    if (ret != pid) {
        printf("[test_waitpid_option_2] waitpid: expected pid[0x%x] but got retval[0x%x]\n", pid, ret);
        return 1;
    }
    if (status != 0x33) {
        printf("[test_waitpid_option_2] waitpid: expected xstate[0x33] but got [0x%x]\n", status);
        return 1;
    }
    printf("[test_waitpid_option_2] Sinocentric Epoch\n");
    return 0;
}

int (*tests[])(void) = {
    test_waitpid,
    test_waitpid_pid_le_0,
    test_waitpid_pid_not_child,
    test_waitpid_option_1,
    test_waitpid_option_2
};

/******************************/
// (7)
void clean_all_childs() {
    int status;
    while (wait(&status) != -1);
}

int all_test() {
    int flg = 1;
    for (int i = 0; i < sizeof(tests) / sizeof(tests[0]); i++) {
        if (tests[i]() != 0) {
            flg = 0;
            printf("[all_test] test[%d] failed\n", i);
        }
        clean_all_childs();
    }
    if (flg) printf("[all_test] all tests passed\n");
    return 0;
}

int main(int argc, char **argv) {
    srand(uptime());
    if (argc > 1) return tests[atoi(argv[1])]();
    else return all_test();
}
```

1. A random number generator. 
2. Test basic `waitpid()` implementation.
3. Test when `pid <= 0`.
4. Test when `pid` is not a child process of current process. 
5. Test `option == 1`.
6. Test `option == 2`.
7. Clean all the child processes.