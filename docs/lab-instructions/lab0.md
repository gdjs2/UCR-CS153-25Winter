# Lab 0 Instructions

## Before you start

1. You don't need to submit ANYTHING for this lab.
2. This is a demo for you to get familiar to the github classroom.
3. Try this step-by-step demo and push your work to see whether you can achieve full score!

## Accept Lab 0

Accept Lab 0 thourgh the link: [https://classroom.github.com/a/tgW8fvsX](https://classroom.github.com/a/tgW8fvsX). Join our classroom and link your github account with your netid. If you don't see your netid in the list, tell me!

## Required System

A Unix-like system!
It can be:

1. Any Linux distribution (e.g., Ubuntu)
2. macOS
3. Windows Subsystem Linux, WSL (Click [here](https://learn.microsoft.com/en-us/windows/wsl/install) to see how to install!). I suggest you all install Ubuntu 24.04. Because QEMU on the repository of Ubuntu 22.04 is with version 6.2, which is incompatible to current xv6. The 8.2 version QEMU in Ubuntu 24.04 works well. 

When you install using powershell command, change the command `wsl --install` to `wsl --install -d ubuntu-24.04` which will change Linux distribution you install. 

## Required Tools

After you get your system set up, you should install some necessary tools for developing xv6. Run the following command in your command line (copy & paste the command just following the $ sign).

If you meet errors when downloading the requirements, check whether you update your package manager's index first. For example, if you are using Ubuntu, you need use `$ sudo apt update` to update your index.

### Debian or Ubuntu
```bash
$ sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu 
```

### Arch Linux
```bash
$ sudo pacman -S riscv64-linux-gnu-binutils riscv64-linux-gnu-gcc riscv64-linux-gnu-gdb qemu-emulators-full
```

### macOS

Install developer tools if you haven't:
```bash
$ xcode-select --install
```

Next, install [Homebrew](https://brew.sh), a package manager for macOS if you haven't:
```bash
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Next, install the RISC-V compiler [toolchain](https://github.com/riscv-software-src/homebrew-riscv):
```bash
$ brew tap riscv/riscv
$ brew install riscv-tools
```

The brew formula may not link into /usr/local. You will need to update your shell's rc file (e.g. [~/.bashrc](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html)) to add the appropriate directory to [$PATH](https://www.linfo.org/path_env_var.html).

Finally install QEMU:
```bash
$ brew install qemu
```

### Test your tools

Run the following command to see whether it can give you the version number for the programs:
```bash
$ qemu-system-riscv64 --version
QEMU emulator version 8.2.2 (Debian 1:8.2.2+ds-0ubuntu1.2)
...
```

And at least one RISC-V version of GCC:
```bash
$ riscv64-linux-gnu-gcc --version
riscv64-linux-gnu-gcc (Ubuntu 13.2.0-23ubuntu4) 13.2.0
...
```
```bash
$ riscv64-unknown-elf-gcc --version
```
```bash
$ riscv64-unknown-linux-gnu-gcc --version
```

## Clone your repository

After you accept the lab, you will get a repository for our lab 0, which has a name `lab0-***`. Clone this repo to your local machine. If you never used Github before (never set a SSH key for your account), please refer to [here](#git-related-questions). 

## Compile 

Get into the directory with your lab0 code.

```bash
$ cd /some/path/to/xv6-riscv-lab0
```

Build and run xv6
```bash
$ make qemu
riscv64-linux-gnu-gcc    -c -o kernel/entry.o kernel/entry.S
riscv64-linux-gnu-gcc -Wall -Werror -O -fno-omit-frame-pointer -ggdb
...
riscv64-linux-gnu-objdump -S user/_zombie > user/zombie.asm
riscv64-linux-gnu-objdump -t user/_zombie | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$/d' > user/zombie.sym
mkfs/mkfs fs.img README user/_cat user/_echo user/_forktest user/_grep user/_init user/_kill user/_ln user/_ls user/_mkdir user/_rm user/_sh user/_stressfs user/_usertests user/_grind user/_wc user/_zombie 
nmeta 46 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 1) blocks 1954 total 2000
balloc: first 767 blocks have been allocated
balloc: write bitmap block at sector 45
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
```

Now, xv6-riscv has been compiled and a bash is running. If you type `ls` at the prompt, you should see output similar to the following:
```bash
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2292
cat            2 3 34264
echo           2 4 33184
forktest       2 5 16184
grep           2 6 37520
init           2 7 33648
kill           2 8 33104
ln             2 9 32920
ls             2 10 36288
mkdir          2 11 33160
rm             2 12 33152
sh             2 13 54728
stressfs       2 14 34048
usertests      2 15 179352
grind          2 16 49400
wc             2 17 35216
zombie         2 18 32528
console        3 19 0

```

There are the files that mkfs includes in the initial file system; most are programs you can run. 

To quit qemu type `Ctrl-a x` (press `Ctrl` and `a` at the same time, followed by `x`)

## Add the First User Program!

Open this directory using any code editor (e.g., vscode). In the directory `/user`, add a file named by `hello-world.c`. 

Copy and paste the following code to the file. Remember to save it.

```c
#include "kernel/types.h"
#include "user/user.h"

int main() {
    printf("hello, world\n");
}
```

Add one line in the file `/Makefile`, just after line 141 and save. 

```makefile
...
	$U/_grind\
	$U/_wc\
	$U/_zombie\
   $U/_hello-world\
```

Now recompile the code using `make qemu`. After you see `init: starting sh`. You can type `ls` to check whether there is a new program naming `hello-world`. If there is, type command `hello-world` to run this program. 

```bash
$ hello-world
hello, world
```

You are done with your work here!

## Push Your Work

You now can push your work to the repository in Github Classroom. You can use the GUI of your editor (e.g., vscode) to do this, or you can use the command line. 

For command line: [https://docs.github.com/en/repositories/working-with-files/managing-files/adding-a-file-to-a-repository#adding-a-file-to-a-repository-using-the-command-line](https://docs.github.com/en/repositories/working-with-files/managing-files/adding-a-file-to-a-repository#adding-a-file-to-a-repository-using-the-command-line)

For vscode: [https://code.visualstudio.com/docs/sourcecontrol/intro-to-git#_staging-and-committing-code-changes](https://code.visualstudio.com/docs/sourcecontrol/intro-to-git#_staging-and-committing-code-changes)

Just focus on Section *Staging and committing code changes* and Section *Pushing and pulling remote changes*


> Hit: Always use `make clean` to clean up the binary files compiled before you commit your change. It can avoid submitting unnecessary files (usually binary and intermediate files). 

## Basic Linux Commands

1. `cd [SOMEWHERE]`: change directory to SOMEWHERE. `cd /` will change directory to root directoty. `cd ..` will change to the upper level directory. `cd dir1` will change to `dir1` under currect directory.
2. `ls` or `ls [SOMEWHERE]`: `ls` is the same as `ls .`, which means list the items in currect directory. You can use a parameter to specify which directory you want to list.

## Basic Git Commands

1. `git clone [URL]`: clone a repository to currect position. 
2. `git add [FILES]`: add some files for tracking.
3. `git commit -c "CMNT HERE"`: commit the changes and append some comment.
4. `git push`: push the commit(s) to the remote repository.

## Git related Questions

### How can I clone a private repository to my local machine?

[https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository)

### What if there is a permission error when I try cloning a repository?

[https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
