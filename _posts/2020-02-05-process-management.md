---
title: 1. 进程管理
date: 2019-06-01
---

进程是操作系统的核心概念，通过进程抽象使得一个物理机（处理器核）被虚拟化成多个，每个进程可以独立拥有一个完整的处理机运行环境。进程抽象依赖于运行环境的切换：CPU 现场的保存与恢复。因此进程切换代码与处理器架构密切相关，也比较难以理解。我们先从进程管理角度分析 `xv6`（这非常容易理解），然后再讨论进程调度中的切换细节问题。

## 1. 调度状态与执行现场

每次时钟中断都将进入到 `xv6` 内核代码，具体是 `trap()` 函数内部，完成 `IRQ_TIMER` 相关的处理（例如 `ticks++`），然后用 `yield()` 让当前进程让出 CPU（切换到其他就绪进程），具体参见 [trap.c#L103](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L103)。

### 1.1 进程控制块 PCB

进程描述符 PCB 在 `xv6` 中是 `proc` 结构体，具体定义请参见 [proc.h#L37](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L37)。proc 结构体记录了： 

1. 有关进程组织的信息：`pid`（进程号）、`parent`（父进程）、`name[]`（进程名）。
2. 进程运行状态信息：`state`（进程的运行调度状态）、`killed`（被撤销标志）。
3. 进程调度切换信息：`tf`（陷阱帧）、`context`（进程的上下文），`chan`（阻塞链表）。
4. 进程的内存映像的信息：`sz`（内存空间大小）、`pgdir`（页表）、`kstack`（内核栈底，创建进程 PCB 时由 `allocproc()->kalloc()` 分配）。
5. 文件系统相关信息：`cwd`（当前工作目录）、`ofile[]`（已打开文件的列表）。

从这里可以看到 `xv6` 的进程亲缘关系组织比 Linux 要简单得多，只有父进程关系，无法知道自己的子进程和兄弟进程。

进程的调度状态在 [proc.h#L35](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L35) 的枚举类型 `procstate` 中列出，分别是 `UNUSED`（未使用的 PCB）、`EMBRYO` （创建中，胚胎状态）、`SLEEPING`（睡眠阻塞中）、`RUNNABLE`（就绪状态）、 `RUNNING`（在 CPU 上运行）以及 `ZOMBIE`（僵尸态）。  

后面我们会看到，`xv6` 使用固定大小的静态数组来记录 PCB，因此未使用的 PCB 必须标为 `UNUSED`，而且系统中创建的进程数受限于该数组的大小。而 Linux 系统则是动态生成 PCB， 并构成链表，因此进程撤销后最终连 PCB 也释放掉而无须 UNUSED 状态，而且进程数目的不会受限于某个静态数组的大小。 

![process states](/xv6-book/img/state.png)

### 1.2 处理器组织管理

由于系统中有多个处理器，因此处理器的数量由全局变量 `ncpu` 记录，其声明 [proc.h#L14](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L14)（其定义在 [mp.c#L15](https://github.com/professordeng/xv6-expansion/blob/master/mp.c#L15)）。有一个全局数组变量 `cpus[NCPU]` 记录所有 CPU 的信 息，由于每个处理器对应的独立元素，因此只要保证各自只访问自己的元素，就不需要进行互斥保护（访问速度比自旋锁保护的变量更快速），这一类变量称为 "每处理器变量“（per-CPU 变量），我们也称它为 ”CPU 私有变量“。

系统中的处理器 `i` 通过 `cpus[i]` 来管理，这是一个 `cpu` 类型结构变量（见 [proc.h#L1](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L1)），用于该处理器上的进程调度管理。相关成员及其作用说明如下：

1. `apicid` 记录本地 `Local APIC ID`（`Intel` 的每个 CPU 都有独立的 `lapic`，每个 `lapic` 有一个 ID，`apicid` 是区分 CPU 的重要标识）。
2. scheduler 指向一个 context 结构体，context 对应于运行在本处理器上的 scheduler 执行现场（`edi` / `esi` / `ebx` / `ebp` / `eip`）。
3. `ts` 是 `taskstate` 结构体（参见 [mmu.h#L106](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L106) ），即 `x86` 的任务状态段的内容，从中可以找到因运行级提升所需的堆栈（例如中断进入内核态，需要切换到内核堆栈）。
4. `gdt[NSEGS]` 是本处理器正在使用的全局描述符表（各处理器差异在于 CPU 私有段）。
5. `started` 表示该处理器是否已经启动，多核系统刚开始有一个引导处理器（核）启 动，然后再其他其他处理器核。
6. `ncli` 记录了关中断的嵌套次数。
7. `intena` 记录了 `pushcli` 之前中断是否使能（打开）。
8. `cpu` 成员指向一个 `cpu` 结构体。  
9. `proc` 成员记录本 CPU 上正在执行的进程。

### 1.3 内核切换现场

我们在 [执行断点](https://neuron.zone/xv6-book/2019/04/11/kernel.html#执行断点) 中已经了解过 ”系统调用/中断断点“ 和 ”切换断点“。这里所谓的内核执行线程，则是内核态 ”切换断点“ 的执行现场。

在进程切换前要保存当前进程的内核态执行现场，并恢复切入进程的执行现场。由于进程切换都是发生在内核态的，而内核态的段寄存器都是相同的（CS 等等），因此无需保存段寄存器。而 `EAX`、`ECX` 和 `EDX` 也不需要保存，因为按照 `x86` 的调用约定是调用者保存，它们保存在该进程的内核堆栈中。`ESP` 也不需要保存，因为 context 本身就是堆栈栈顶位置，而进程 PCB 中 `proc->context` 成员就指向 context 就是堆栈对应的位置。于是 `xv6` 中的进程内核断点执行现场 context 只有成员 `edi`、`esi`、`ebx`、`ebp`、`eip`（参见 [proc.h#L16](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L16)）。 

例如，某进程用户态代码执行 `yield()` 系统调用而希望让出 CPU，本进程的内核堆栈中形成的 `trapfeame`。然后随着内核代码的执行和函数调用的嵌套，形成更多层次的函数调用栈，直至 `swtch()` 为止，最后在切换前在堆栈保存现场 context。

### 1.4 proc.h

`proc.h` 文件中定义了 `cpu` 结构体，用于描述个处理器核；定义了全局变量 `cpus[NCPU]` 记录了全部处理器核的信息，以及全局变量 `ncpu` 用于记录处理器核数；声明了外部定义的两个 CPU 私有变量，`cpu` 用户获取自己所在的 CPU 核的指针，`proc` 变量用于获取当前正在运行的进程指 针；定义了 context 用于结构体，用于记录内核切换断点的现场；用于表示进程调度状态的枚举值 `procstate`；最后是描述 `xv6` 进程 PCB 的 `proc` 结构体。

### 1.5 proc.c

`proc.c` 是 `xv6` 进程管理的核心代码。包括一些全局性的变量，例如 `ptable` 用于记录管理所 有进程，其中 `ptable.proc[NPROC]` 数组用于记录所有进程的 PCB；`initproc` 是 `init` 进程的 PCB。

## 2. 进程控制

`xv6` 中使用 `ptable` 中的静态数组 `ptable.proc[]` 来记录和组织进程，请见 [proc.c#L10](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L10)。该数组最大可以记录 `NPROC`（64，参见 [param.h#L1](https://github.com/professordeng/xv6-expansion/blob/master/param.h#L1)）个数的进程，并且由 `lock` 自旋锁保护。该自旋锁通过 [proc.c#L23](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L23) 的 `pint()` 完成初始化（也是 `pint()` 的唯一用处）。 第一个进程的 PCB 由 [proc.c#L15](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L15) 的 `initproc` 指针变量指出。全局变量 `nextpid` 是下一个可用的进程号，并初始化为 1。（貌似 `xv6` 对进程号不重复使用，单项递增，可能考虑到整数范围对 `xv6` 实验而言有足够大）

### 2.1 进程创建与撤销

创建进程时需要分配一个空闲的 PCB，该工作由 [proc.c#L68](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L68) 的 `allocproc()` 完成。[proc.c#L81](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L81) 在进程 PCB 数组 `ptable.proc[]` 中查找未使用的一项。

如果找到空闲 PCB，则 [proc.c#L89](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L89) 将其状态修改为胚胎态 EMBRO，将其进程号 `pid` 设置为 `nextpid`（同时 `nextpid` 自增 1）。注意，进程号 `pid` 和 PCB 在数组 `ptable.proc[n]` 中的位置（数组元素索引号 n）没有必然联系。然后 [proc.c#L94](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L94) 通过 `kalloc()` 分配内核栈空间，`p->kstack` 记录堆栈内存区间的起始位置（低端地址），再进一步将内核陷阱帧 `p->tf` 和堆栈指针 `sp` 指向堆栈内存空间的最高地址减去 `trapframe` 的空间。

然后在堆栈中分配 4 字节的 `trapret` 和 `context` 结构体，这样伪造成一个进程从用户态进入内核态的 `trapframe`，然后在伪造出进程切换断点在 context 中，这样就可以通过进程切换 “返回” 到进程应该执行的位置（例如可能是程序第一个指令，或者 `fork` 之后的下一条指令）。 

早期的 Linux 是将 PCB（`task_struct` 结构体）和内核堆栈放到一起，找到了进程控制块后根据固定偏移量也就找到了内核堆栈。后来的 Linux 堆栈也是和 PCB 分离的，类似于 `xv6` 这种 方式。

### 2.2 第一个进程 initcode

在 `xv6` 启动过程中 `main()->userinit()` 创建第一个用户态进程 `initcode`。`userinit()` 见 [proc.c#L118](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L118)，它将完成第一个用户进程 `init` 的创建工作。这个进程镜像很快随着 `initcode` 的执行而通过 `SYS_exec` 系统调用替换成磁盘上的 `/init` 进程影像，启动 `sh` 程序（如果 `sh` 程序结束，则再生成一个 `sh`）。 

首先 [proc.c#L126](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L126) 内存管理部分刚讨论过的 `allocproc()`，分配一个空闲的 PCB 并初始化相应的内核堆栈，并使用专门的全局变量 `initproc` 来记录这个进程。然后 [proc.c#L129](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L129) 通过 `setupkvm()` 给 `init` 进程创建初始页表，由于只映射了内核空间因此函数名使用 `kvm`（`kernel vm`），因此还没有用户态页表（不能访问用户态空间）。由于代码很短，只需要一个页就将代码数据和堆栈包含了，因此 `p->sz = PGSIZE`。

接着通过 `inituvm()` 完成用户空间的建立，由于 `init` 的代码已经在内核镜像 `kernel` 中，随着启动过程装载到 `_binary_initcode_start` 地址，因此只需要新分配一个页帧，然后将该页帧映射到 0 地址，再将 `init` 代码拷贝到该地址即可。

然后 [proc.c#L133](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L133)，自行（而不是因中断）设置了一个 `trapframe` 结构 `p->tf`，造成等效于 “好像曾经” 从用户态经过中断而形成的 `trapframe`，也就是说一旦利用这个 `trapframe` 进行 `iret` 返回就会返回到设定好的 `init` 进程用户态断点处（伪造出来的断点），即 `eip` 指向的 0 地址，正是 `initcode` 的第一条指令位置。从这里可以看出，`xv6` 的进程布局与 Linux 安排不同，Linux 的进程入口第一条指令在 `0x40000` 附近。 

最后设置进程名 `p->name` 为 `initcode`、修改当前工作目录为 `/` 并将进程调度状态设置为 `RUNNABLE`（就绪）。 

创建 `init` 进程的过程和 `fork()` 函数完成的操作有一些相似的地方，但是 `init` 进程由于是第 一个进程，无法通过拷贝的方法来创建进程的内存镜像。

### 2.3 fork 

除了第一个进程外，其他进程都需要通过 `fork` 系统调用来产生。[proc.c#L177](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L177) 的 fork() 实现了 Unix 概念中的子进程创建操作。类似创建 `init` 进程，首先需要 `allocproc()` 分配空闲的 PCB 记录在 `np` 变量中。 

然后 `copyuvm()` 拷贝父进程镜像作为子进程镜像（含内核空间和用户空间），子进程镜像本质上就是页表的设置 `np->pgdir`。然后复制进程空间大小 `np->sz=proc->sz`（这个 `proc` 变量是 CPU 私有变量，指向当前在运行的那个进程，即父进程）。设置父进程 `np->parent=proc` 为当前进程。任务状态段也进行复制 `*np->tf=*proc->tf`，但是子进程返回值为 0，于是 `np->tf->eax=0`。 还要拷贝已打开的文件 `np->ofile[]` 数组。拷贝当前工作路径 `np->cwd=idup(proc->cwd)`。拷贝进程名 `np->name`。设置进程状态 `np_state=RUNNABLE`。最后将 `pid` 通过函数返回值 `eax` 返回给父进程。 

父进程返回值为子进程 `pid`，而子进程的返回值为 0，因为子进程被调度的时候从根据 `trapframe` 的内容返回到 `fork()` 函数的下一条指令处往下继续运行。又因为 `eax` 被设置为 0，造成好像是子进程也执行了 `fork()` 代码并返回 0 的假象，实际上子进程直接就是从 `fork()` 后的下一条指令开始运行的。

从 `fork()` 拷贝的资源来看，`xv6` 的进程资源相对于 Linux 来说要少的多，主要就是内存空间和所打开的文件。

### 2.4 forket()

子进程的执行起点，将返回到用户态，发出 `fork()` 函数调用的的下一条语句。

### 2.5 kill

撤销指定 `pid` 的进程使用 `kill()` 函数，定义于 [proc.c#L476](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L476)。为了找到将要撤销的进程，需要扫描进程列表 `ptable.proc[]` 数组，逐个检查该进程号是否和传入的 `pid` 值相同。`kill()` 在撤销进程操作过程中主要是负责将 `p->killed` 标志置位，并不负责进程状态的修改。只有在被撤销进程处于睡眠 `SLEEPING` 状态时，将其状态修改为 `RUNNABLE`，只有经过 `RUNNALBE` 才能之进入 `RUNNING` 最后进入到 `ZOMBIE` 状态。 

真正的撤销操作需要 `exit()` 函数完成，这只有在返回用户态之前才能执行。

### 2.6 exit

进程结束时完成 `exit()` 操作，请参见 [proc.c#L224](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L224)。首先，`init` 进程是不允许退出的，索引当系统发现执行 `exit()` 函数的是 `init` 进程则通过 `panic()` 打印警告信息 `init exiting`。 [proc.c#L237](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L237) 将关闭所有打开的文件，即遍历 `proc->ofile[]` 数组，对每一个文件执行 `fileclose()`，并通过 `iput()` 释放对当前工作目录的索引节点的占用（[proc.c#L246](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L246)）。

由于自己要退出了，父进程通常在 `wait()`，所以需要用 `wakeup1()` 唤醒父进程。 

接着将自己的子进程移交给 `init`。由于 PCB 中没有子进程的信息，因此只能遍历所有进程，看看它们谁的父进程指向自己（以此判定自己的子进程）。可以看出，Linux 中有完善的进程亲子关系组织，因此子进程的查找不需要这种低效的方法。如果子进程已经处于 `ZOMBIE` 状态则还需要唤醒 `init` 进程进行最后的清理工作。

最后将自己的状态设置为 `ZOMBIE`（等待父进程做最后的检查和清理），并通过 `sched()` 切换到其他就绪进程。 

### 2.7 增减用户内存空间（分配和释放）

进程如果要分配内存而改变自己的内存镜像，则需要用 `growproc()` 函数。[proc.c#L156](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L156) 的 `growproc()` 用于在用户空间分配 n 个字节。其中 n 为正数时时进行扩展（分配内存），而 n 为负数时进行收缩（释放内存）。分配操作是通过 `allocuvm()` 完成，而释放操作时通过 `deallocuvm()` 完成。`allocuvm()` 和 `deallocuvm()` 已经在内存管理的部分进行了分析。 

## 3. 进程控制

### 3.1 sleep

[proc.c#L415](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L415) 给出了 `xv6` 进程睡眠阻塞的 `sleep()` 函数。需要传入阻塞队列 `chan` 和自旋锁 `lk`，其中 `chan` 根据事件的不同而不同，例如可以是一个 `buf` 缓冲区（在 `iderw()` 中）。 也就是说，`xv6` 并没有使用专门的阻塞队列这样一个数据结构，而是通过将等待相同事件的进程 `proc->p` 指向相同的的数据对象（即地址）来识别的。

[proc.c#L438](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L438) 将进程挂入到阻塞队列上，并将状态修改为 `SLEEPING`，通过 `sched()` 切换到其他进程。 

如果 `lk` 不是 `ptable.lock` 的时候，在修改进程状态之前还要对 `ptable.lock` 进行加锁。此时 将同时持有 `lk` 和 `ptable.lock`，因此可以将 `lk` 解锁，此时即使有其他进程尝试 `wakeup(chan)`， 也会因为没有 `ptable.lock` 而无法开展 `wakeup` 操作，必须等我们将 `ptable.lock` 释放。 

### 3.2 wakeup

如果需要唤醒阻塞队列上的所有进程则使用 `wakeup()`，它在 [proc.c#L467](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L467) 。它在获得进程 PCB 数组 `ptable` 的锁之后，进一步 `wakeup1()` 来完成唤醒工作。 具体操作是通过扫描所有进程，看是否在指定的阻塞队列 `chan` 上睡眠（状态为 `SLEEPING`）， 如果是则将其状态修改为 `RUNNABLE`（`RUNNABLE` 状态将忽视 `p->chan`）。 

将 `wakeup` 操作分成两部分的原因是，有时候已经持有了 `ptable.lock` 锁，这时只需要直接调用 `wakeup1()` 即可。

从这里也可以看出，确实不存在睡眠阻塞队列，而仅仅是靠等待相同的事件来表示。 

### 3.3 yield

进程需要让出 CPU 时执行 `yield()` 函数，`xv6` 的 `yield()` 函数在 [proc.c#L384](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L384)。所作操作很简单，就是将状态设置为 `RUNNABLE`（本来是正在执行的 `RUNNING` 状态），然后执行 `sched()` 切换到其他进程。

时钟中断 `tick` 处理程序中，将会执行 `yield()` 将本进程让出 CPU，并调用 `sched()` 从而完成进 程切换。

### 3.4 wait

父进程等待子进程退出时将执行 `wait()` 操作，`xv6` 的 `wait()` 函数在 [proc.c#L270](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L270)。 就如前面所示，子进程的查找只能通过遍历所有进程并根据其父进程是否指向自己来判定。对找到的子进程，如果其状态为 `ZOMBIEZ`（执行了 `exit` 系统调用之后的状态），则需要对僵尸子进程做最后的撤销工作。

如果没有子进程退出，则通过 `sleep()` 进入睡眠阻塞状态（当子进程退出而执行 exit()时会 唤醒父进程）。

### 3.5 procdump

如果在 `xv6` 的 shell 命令行中输入 `Ctrl + p` 将显示当前进程的信息列表。`procdump()` 请参见 [proc.c#L499](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L499)。该函数对进程控制块数组 `ptable.proc[]` 进行扫描，显示 `used` 的进程 PCB 的关键信息。 

```bash
1 sleep  init 80103e27 80103ec7 80104879 80105835 8010564f
2 sleep  sh 80103dec 801002ca 80100f9c 80104b62 80104879 80105835 8010564f
```

从代码中可以看出每一行的前三列信息包括：进程号、状态、进程名（可执行文件名），因此上述信息表明有 1 号进程 `init` 处于睡眠 `SLEEPING` 状态，2 号进程 `sh` 处于睡眠 `SLEEPING` 状态。后面的数字是各级调用返回地址（`EIP` 值），最右边是最底层的函数，如果调用次数多于 10 层则只显示 10 层，不足 10 层按实际调用层数显示。

代码中有一个 `NELEM` 宏，用于计算一维数组的元素个数（`defs.h` 中定义）。 

## 4. 进程调度

进程调度主要涉及两个方面，一个是调度器，一个是进程切换代码。`xv6` 的进程调度器比较简单，只是简单扫描就绪进程并选择下一个可运行的进程；而切换代码则略显复杂，涉及切换现场的保存和恢复，而且和任务状态段 `TSS` 的细节有关。 

### 4.1 scheduler

[proc.c#L314](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L314) 是 `scheduler()` 函数，每个 CPU 在启动后将调用 `scheduler()`，该函数是一个无限循环并不返回，不断选择下一个就绪进程并且换过去执行。在主 CPU 启动过程中 `main()->mpman()->scheduler()`，以及其它处理器的 `mpenter()->mpmain()->scheduler()` 而启动。 

`scheduler()` 代码的安排如下：外层是一个无限循环；内层循环将扫描遍历 `ptable->proc[]` 数组，如果找到一个 `RUNNABLE` 的就绪进程，则 [proc.c#L339](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L339) 切换过去执行。其中变量 `proc` 是每 CPU 变量，记录当前 CPU 上正在执行的进程。从此可以看出其调度算法实际上就是轮转算法，按照进程在 PCB 表 `ptable` 中的次序轮转（而不是根据进程号轮转）。

进程切换过程则通过 `switchuvm()` 完成进程 `TSS` 切换和用户空间的切换，使得切入进程镜像可见。然后设置切入进程状态 `p->state=RUNNALBE`。`swtch()` 换成内核运行现场的切换（恢复切入进程的切换断点的现场），从此执行被调度的进程。

需要注意的是：只有到被调度进程因各种原因让出 CPU 而执行 `sched()` 的时候，才会从 `swtch()`返回。一旦返回，则 `scheduler()` 将要重新接管 CPU，通过 `switchkvm()` 将页表恢复成 `scheduler()` 自己的页表（仅在内核空间有映射），最后将 `proc=0` 表示当前没有运行用户进程， 然后进入下一次调度循环选取下一个就绪进程。

由于 `scheduler()` 是在内核态运行的，因此用户空间的切换 `switchuvm()` 执行过后，对 `scheduler()` 本身的运行没有任何影响，运行的仍是内核的 `scheduler()` 代码。

### 4.2 sched

`sched()` 函数是用户进程切换到 scheduler 执行流时所调用的代码，scheduler 切换到用户进程是在 `scheduler()` 调用 `swtch()` 完成的。[proc.c#L358](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L358) 是 `sched()` 函数，它在进程主动睡眠时、时间片到、IO 读写需要阻塞、主动 `yield()` 放弃 CPU 等时机，都会被调用。`sched()` 的作用是从当前进程切换到 scheduler 执行流，从而让出 CPU。 

需要注意 `sched()` 调用的条件，首先必须持有 `ptable.lock` 锁。

当前进程不能仍处于 `RUNNING` 状态，按理应该处于 `RUNNABLE`、`SLEEPING`、`ZOMBIE` 状态。例如

1.  执行 `sleep()` 睡眠操作时，必须先将当前进程状态修改为 `SLEEPING`，然后再调用 `sched()`。
2. 执行 `exit()` 退出操作时，需要先将当前进程状态修改为 `ZOMBIE` 后再调用 `sched()`。
3. 执行 `yield()` 让出 CPU 时，当前运行进程的状态修改为 `RUNNABLE`，再调用 `sched()`。

`sched()` 和 `scheduler()` 的切换过程在 “切换断点” 的处理上是相同的，都是通过 `swtch()` 完 成 context 现场的切换。 

## 5. 执行流的切换

这里的 “执行流切换” 指的是：

1. 从 scheduler 的执行流切换到切入进程的执行流的过程。
2. 从换出进程的执行流切入到 scheduler 的执行流。

执行流的切换也将伴随着空间的切换，即页表的切换。 

进程执行流切换时，正在执行的待换出进程是处于运行 scheduler 状态，也就是说页表是 scheduler 的页表。所以换出和切入的两个进程是在内核态执行现场的切换。因此，读者需要完整理解这个过程，就首先需要了解从用户态进入到内核态的系统调用和中断过程。

### 5.1 进程镜像的切换

为了从 scheduler 执行流切换到切入进程，需要用 `switchuvm()` 完成进程镜像切换，此时 scheduler 执行流使用的是切入进程的镜像（段表和页表）。切换段表的原因是每个进程有自己私有的 `SEG_KDATA`（用于保存 CPU 私有变量）以及 TSS 中保存的切入进程内核栈起点指针。 然后经过 `swtch()` 切换到切入进程执行流。 

从换出进程返回到 scheduler 执行流的时候，则通过 `switchkvm()` 完成进程镜像切换到调度器的内存镜像，仅切换页表，因为 scheduler 执行流仍可以访问 CPU 私有变量 `proc` 获得当前进程 PCB，但 scheduler 无需 TSS 中的信息，所以仍可以使用该进程的段表和页表。

### 5.2 swtch

`swtch()` 是在内存镜像已经完成切换之后才执行的，负责：

1. CPU 运行现场 context 切换。
2. 内核堆栈切换。
3. 指令 `eip` 的跳转 。

`swtch` 虽然是汇编代码片段，但是它相当于 C 语言函数 `void swtch(struct context **old, struct context *new)`，其参数按照 C 函数规范在堆栈中指定的位置。

这里切换代码中没有出现 context 中的 `eip` 成员，它是硬件操作的，通过函数调用的 call 指令压入 `eip` 堆栈形成 context 的 `eip` 成员，同理对于新的 context 在执行 `ret` 时会从堆栈中弹出 `eip`。  

通过 `swtch()` 完成切换断点的现场切换，用切入进程的切换断点现场替代换出进程的切换断点现场。[swtch.S#L20](https://github.com/professordeng/xv6-expansion/blob/master/swtch.S#L20) 进行堆栈切换，也完成了 context 的切换，因为此时的 `esp` 指向的就是 context 内容（请回顾前面提到的进程内核态运行现场存储于 context）。 

下面我们来分析 `swtch()` 的 `ret` 执行前后是如何完成执行流的切换的。首先要认识到的是，被换出进程执行流的断点是在 `sched()->swtch()` 的下一条语句，也就是断点的 `EIP` 指向
 `cpu->intena=intena;`。因此一个进程在执行 `sched()` 的过程中，不是一次性完成的，在断点 处将会切换到 `scheduler()` 执行流，而恢复运行的时候则是从 `scheduler()` 执行流返回到这个断点 `cpu->intena=intena;`  继续运行。其次要认识到 `scheduler()` 的执行流断点是在 `scheduler()->swtch()` 的一下条语句，也就它的断点 `EIP` 指向的是 `switchkvm();`。因此前面 提到过 `scheduler()` 函数不是一次性完成的，在断点处将会切换到调度的就绪进程，而恢复运行 时则从那个进程的执行流返回到断点 `switchkvm()` 继续运行。

### 5.3 返回到用户态

进程在内核态完成切换，最终还要返回到用户态断点，才能恢复用户进程继续运行。我们还是借助于内核栈的 `trapframe`，下面以 B 进程的堆栈为例。当内核代码逐层返回后最终将遇到 `trapframe` 帧，而无论是系统调用还是中断返回，都会执行 `trapret` 代码。也就是说 B 进程在内 核态经过 `sleep()` 函数返回，最终执行到 `trapret` 代码（参见 [trapasm.S#L23](https://github.com/professordeng/xv6-expansion/blob/master/trapasm.S#L23)），它将从内核堆栈恢复现场：`popal` 弹出通用寄存器、然后弹出并恢复原来的 `gs`、`fs`、`es`、`ds`，弹出 `trapno` 和 `errcode`，最后执行 `iret`，此时将返回到 `trapframe`，一开始就记录的用户态断点（CS、EIP 和 EFLAGS），返回到用户态是从 `ring0` 转到 `ring3`，因此会从堆栈 `trapframe` 中恢复用户态堆栈 `SS` 和 `ESP`（当用户态经系统调用 / 中断进入内核态，发生 `ring3`→`ring0` 的运行级提升时，堆栈要切换到内核栈是借助 `TSS` 中的 `SS0` 的 `ESP0` 由硬件自动完成的）。 

### 5.4 进程调度时堆栈切换问题

进程放弃在 CPU 上运行的时候，调用 [swtch.S#L3](https://github.com/professordeng/xv6-expansion/blob/master/swtch.S#L3) 的 `swtch` 代码片段来保存自己的切换断点执行环境（struct context），并且换到 scheduler 的执行环境。

读者需要将中断的进入和退出操作，与进程运行环境切换操作的进行关联，才能构成内核栈的全景认识。

scheduler 的执行环境 context 记录在 `cpu->scheduler` 上，其中 `cpu` 结构体请参见 [proc.h#L1](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L1)，因此任何一个进程都可以通过 `cpu->scheduler` 找到对应的执行环境完成第一步切换步骤。而进入到 scheduler 之后，将会扫描进程列表 `ptable->proc[]` 找到就绪进程，并再通过一次 `swtch()` 完成执行环境的切换。