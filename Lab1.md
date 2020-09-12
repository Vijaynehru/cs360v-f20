## Project-1

In this project, you will implement a few exciting pieces of a paravirtual hypervisor. You will use the JOS operating system running on QEMU for this project. Check the [tools page](https://github.com/vijay03/cs360v-f20/blob/lab1/tools.md) for an overview on JOS and useful commands of QEMU. The project covers bootstrapping a guest OS, programming extended page tables, emulating privileged instructions, and using hypercalls to implement hard drive emulation over a disk image file. You will work on them over the next 3 or 4 lab assignments.

## Background

This README series contains some background related to project 1. Reading this document will help in understanding what exactly you are implementing on a high level as you work on Project-1.

This README series is broken down into 4 parts:
1. [Bootloader and Kernel](https://github.com/vijay03/cs378-f19/blob/master/bootloader.md) which will help you in the understanding first part of the project, namely what happens when you boot up your PC (in our case, create or boot the guest).
2. [Virtual Memory](https://github.com/vijay03/cs378-f19/blob/master/virtual_memory.md) which will help you in understanding the second part - where we transfer the multiboot structure from the host to the guest, for the guest to understand how much memory it is allocated and how much it can use. This part also contains details about segmentation and paging, which are a good background for the project in general.
3. [Environments](https://github.com/vijay03/cs378-f19/blob/master/environments.md) which will help you in understanding what exactly is an environment, and some details about the environment structure which is used in sys_ept_map() and the trapframe structure.
4. [File System](https://github.com/vijay03/cs378-f19/blob/master/file_system.md) which will help you in understanding the second part of the lab, where we handle vmcalls related to reading and writing of data to a disk.

## Lab-1

For Lab-1, you will first set up your working environment and then implement code for making the guest environment.

## Getting Started

You may use your laptops / computers for this project. Please enable qemu-kvm on your machines. Alternatively you can also use any of the following (gilligan) CS machines. As you need access to KVM module for this project, you cannot use other CS machines.
- ginger
- lovey
- mary-ann
- skipper
- the-professor
- thurston-howell-iii

For lab-1, you will use a virtual machine with Ubuntu 16.04 operating system. Follow the instructions below for
1. Setting up a VM and other essentials
2. Running JOS code for project-1

#### 1. Setting up a Virtual Machine and Other Essentials

1. Download the [VM image](http://www.cs.utexas.edu/~soujanya/project1-vm.qcow2) (8.9 GB) on CS gilligan machines or your personal laptops (with QEMU and KVM enabled).
```
$ wget http://www.cs.utexas.edu/~soujanya/project1-vm.qcow2
```

2. Now start up a VM that listens on a specific port using the following command. To avoid contention over ports, use `<port-id> = 5900 + <team-number>`. For example, if your group-id is 15, your port-id will be 5915.
```
$ qemu-system-x86_64 -cpu host -drive file=<path-to-qcow2-image>,format=qcow2 -m 512 -net user,hostfwd=tcp::<port-id>-:22 -net nic -nographic -enable-kvm
```

3. On another terminal, connect to the VM using the following command. On connecting, enter the password as `abc123`.
```
$ ssh -p <port-id> cs378@localhost
```

5. Copy your public *and* private ssh keys from the CS lab machine or from your local machine into the VM.
Alternatively, you can generate a new key-pair on the VM using `ssh-keygen -t rsa`. You should send the public key in the VM to the TAs.
```
$ scp -P <port-id> $HOME/.ssh/id_rsa.pub $USER@localhost:~/.ssh/id_rsa.pub
$ scp -P <port-id> $HOME/.ssh/id_rsa $USER@localhost:~/.ssh/id_rsa
```

6. You will now be able to clone the project code `project-1.tar.gz` and access it from the VM. We will provide additional instructions later on how to use gitolite for project-1.
```
$ wget http://www.cs.utexas.edu/~soujanya/project-1.tar.gz
$ tar -zxf project-1.tar.gz
$ cd project-1
```

7. Verify that you have gdb 7.7 and gcc 4.8. Also cross check that you have python-3.4 installed or in your $HOME directory. In case you need to install any of them, follow the instructions on the [installations](https://github.com/vijay03/cs360v-f20/blob/lab1/installation.md) page. Note that to exit from QEMU VM press `Ctrl a` followed by `x`.


#### 2. Bootstrapping JOS VMM

The JOS VMM is launched by a fairly simple program in user/vmm.c. This application calls a new system call to create an environment (similar to a process) that runs in guest mode (sys_env_mkguest). Once the guest is created, the VMM then copies the bootloader and kernel into the guest's physical address space, marks the environment as runnable, and waits until the guest exits.

You will need to implement key pieces of the supporting system calls for the VMM, as well as some of the copying functionality.

Compile the code using the command:
```
$ make clean
$ make
```
Please note that the compilation works with gcc version <= 5.0.0. The Makefile specifically uses GCC 4.8.0, which is present in the gilligan lab machines. Please install gcc-4.8 if you haven't already to avoid the bootloader too large error.

You can try running the vmm from the shell in your guest by typing:
```
$ make run-vmm-nox
```
This will currently panic the kernel because the code to detect vmx and extended page table support is not implemented, but as you complete the project, you will see this launch a JOS-in-JOS environment. You will see the following error:
```
kernel panic on CPU 0 at ../vmm/vmx.c:65: vmx_check_support not implemented
```

## Coding Assignment (Making a Guest Environment)

The JOS bookkeeping for sys_env_mkguest is already provided for you in kern/syscall.c. You may want to skim this code, as well as the code in kern/env.c to understand how environments are managed. A major difference between a guest and a regular environment is that a guest has its type set to ENV_TYPE_GUEST and has a VmxGuestInfo structure and a vmcs structure associated with it.

The vmm directory includes the kernel-level support needed for the VMM--primarily extended page table support.

Your first task will be to implement detection that the CPU supports vmx and extended paging. Do this by checking the output of the cpuid instruction and reading the values in certain model specific registers (MSRs).

Read Chapters 23.6, 24.6.2, and Appendices A.3.2-3 from the [Intel Manual](http://www.cs.utexas.edu/~vijay/cs378-f17/projects/64-ia-32-architectures-software-developer-vol-3c-part-3-manual.pdf) to learn how to discover if the CPU supports vmx and extended paging.

Once you have read these sections, implement the vmx_check_support() and vmx_check_ept() functions in vmm/vmx.c. You will also need to add support to sched_yield() in kern/sched.c to call vmxon() for launching a guest environment.

If these functions are properly implemented, an attempt to start the VMM will not panic the kernel, but will fail because the vmm can't map guest bootloader and kernel into the VM. The error will look something like this:
```
Error copying page into the guest - 4294967289
```

## Submission Details

Submissions will be handled through gitolite. We will send out an announcement once gitolite is set up.

Please use gitolite to modify the source code, commit, and push your code changes. Commits made until midnight on the day of the submission deadline will be used for grading.

## Contact Details

Reach out to the TAs in case of any difficulties.