## Undsertanding Linux Kernel

[TOC]



### 1 **Introduction**

1. 进程地址空间：进程执行指令的地方，指进程允许被访问的内存地址集合

2. 在内核里面，有一个叫做`kernel control path`的名字可以理解为用户态的进程/线程。

3. 下面的事件可以触发CPU的`interleaves the kernel control path`，即进入内核态的意思:

   - 用户态进程触发系统调用
   - CPU检测到exceptions，exception指系统硬件发生错误，page缺页等等
   - 硬件触发interrupt
   - 更高优先级的进程通过中断的方式抢占了当前CPU

4. 在内核里面，通过下面的技术来进行同步：

   - 禁用抢占模式，多CPU场景下无效
   - 禁用interrupt，多CPU场景下无效
   - 通过信号量来同步，最常见的做法，单CPU和多CPU均适用。
   - 自旋锁：对于临界区操作时间较短的场景比较有优势，信号量本身的开销相对来说就比较大。

5. 进程组和session的概念：

   > 				session一般来讲是启动时候的shell，session里面包含多个进程组，进程组里面又包含多个进程，进程可以将自身移动到同一个session下面的其他进程组里面，但是不可以移动到其他session里面的进程组，即使是root用户也是不可以的。

6. Linux使用Virtual memory的优点：

   - 进程可以并发的执行
   - 可以运行需求比实际物理内存大的应用程序
   - 进程可以执行一部分代码被读取到内存中的程序
   - 每一个进程可以访问物理内存的一个子集
   - 进程可以分享library或者program的内存镜像
   - 进程可以被重新分配，指进程可以被放在物理内存的任何位置
   - 应用开发者可以开发机器无关的代码，不需要关心物理内存实际上是怎么组织的。

### 2 **Memory Addressing**

1. 三种地址的转换：`logical address ->（通过segmentation unit转换） linear address(virtual address) ->（通过paging unit转换） physical address`

   `logical address：`包含两部分：16位的Segment selector和32位的offset，其中Segment selector用来快速查找segment descriptor

   `linear address：`即通常理解中的虚拟地址

   `physical address：`最终的物理地址

   ![地址转换](3511F14575174B0EAF81200A10720835)

2. 目前的主流操作系统基本都是使用分页(`Paging`)机制，而不是分段(`Segment`)机制，只有老的处理器，Linux才会使用分段机制。

3. 分段机制，负责将logical address转换为linear address：

   - 有专门的寄存器来存储`segment selector`，分别为：

     > cs：用来存储code segment
     >
     > ss：用来存储program stack segment
     >
     > ds：用来存储data segment
     >
     > 剩余的四个是通用寄存器，可以存储任意种类的segment

   - `Segment Descriptor`主要被存储在GDT和LDT里面，其中GDT是公用的，LDT是进程独享的。

     > 当进程转换地址的时候，在快速模式下，因为`segment selector`和`segment descriptor`都存储在寄存器中，所以可以直接找到；在普通模式下，会通过`segment selector`在GDT或者LDT中查找到`segment descriptor`，然后通过特定计算公式得到`linear address`，最终得到物理地址。下图为转换的示意图：

     ![segment查找方式](5609A4825BC94DB7B5DC02ED683A99C5)

4. 分页机制，负责将linear address转换为physical address：

   - 为了节省page table的条目，旧的x86结构采用两层 page table，而目前的x64架构采用4层page table，以x86结构为例子，一个32bit的linear地址包含下面三个部分：

     > Directory：高10位      
     >
     > Table：中间10位     
     >
     > Offset：低10位，这样子每个page table里面就包含了2^10 个条目

   - TLB: 存储物理地址到PAGE TABLE的映射，方便高速找到映射关系。当CPU里面的cr3寄存器被更改后，硬件会自动将TLB里面的条目设为无效，因为这表示着新的的page table已经上线，而TLB里面的内容已经过时。
   - Linux当前为了兼容32位和64位，以及PAE模式，采用四层分组，从顶向下分别为：

     > Page Global Directory，Page Upper Directory， Page Middle Directory， Page Table

     Linux自动化的将linear address转换为physical address使得下面的目标变得可行：

     > a、给每一个进程分配不同的物理地址空间，并能有效防止寻址错误。
     >
     > b、区分page（一组数据）和page frame（主内存上的物理地址），这样子可以允许将page映射到page frame上，并且可以保存在磁盘上，后面需要的时候在将page映射到不同的page frame。这种机制是实现虚拟内存的基本。

   - 当前内核会将自身load到RAM的第一帧处，有如下原因：

     > a、最开始的page frame被BIOS使用，用来商店，自检等用途
     > b、0x000a0000 - 0x000fffff通常被用来做BIOS routines和ISA图形卡。
     > c、这些地址还有可能被特定的计算机模型使用，比如IBM的ThinkPad就使用了0xa0到0x9f的page frame

   - kernel初始化page table的方式，有两个阶段，分别为：

     > a、首先创建一个有限的地址空间（非最终的）来初始化kernel 代码，data segments，初始page table和128KB的动态数据结构，这个最小化的地址空间只是刚好能够适合安装kernel，并且初始化其核心的数据结构。
     > b、第二阶段，kernel使用所有的RAM来正确的创建page table

   - 初始化page global directory的函数paging_init做的事情如下：

     > a、调用pagetable_init函数来设置page table entries
     > b、将swapper_pg_dir（用来存储page global directory的变量）的物理地址写入到cr3寄存器
     > c、如果CPU支持PAE并且kernel编译的时候打开了PAE选项，那么设置PAE flag到cr4寄存器
     > d、调用__flush_tlb_all()来清空所有的TLB entries。

   - kernel为了提高cpu cache的命中率，做了如下事情：

     > a、 最高频使用到的数据结构被尽可能的放在cache line的最低位处，这样子就能尽量保证这些数据结构在同一个cache line里面。
     > b、 当分配一组比较大的数据结构时，kernel尽量将这些存储在内存中，这样子就可以更加统一的使用cache line了。

   - 对于以下情况，kernel会避免进行TLB的flush操作

     > a、当在两个普通的进程间进行切换时，而这两个进程使用了同一组page table，此时，不需要进行flush操作。
     > b、当从普通的进程切换到内核 thread的时候，因为内存thread没有他们自己的page table，因此，这些内核thread使用普通进程的page table是最高效的方式。
     

### 3 **Processes**

- 父进程和子进程共享相同的代码，但是地址空间是拷贝出来的，因此stack和heap是相互独立，互不影响。
- Linux2.6用不同的方式实现了`runqueue`，可以在常量时间里找到合适的进程来运行。

- `pid->process descriptor`的映射是通过哈希表的方式来实现的，但是需要四个哈希表，为什么？因为pid也有四种类型，分别为：

  ![](C:\Users\seancheer\Desktop\Programming\读书笔记\res\pid分类.png)

- 对于哈希表，有时候我们需要找到一组进程，比如tgid哈希表，通过tgid找到该进程组里面的所有进程，为了实现该功能，内核使用了一个数据结构来实现该方式，下面的数据结构：

  ![](C:\Users\seancheer\Desktop\Programming\读书笔记\res\hash表数据结构.png)

  > 其中, pid_chain表示哈希冲突的链表，pid_list表示一组进程，比如如果是pgrp哈希表，那么该变量表示属于同一个组的所有pid。

  下图为寻址方式：

  ![](res/hash表寻找过程.PNG)

- > Linux的`runqueue lists`会将所有`TASK_RUNNING`状态的进程链接起来，除了以下情况：
  > 1、`TASK_STOPPED`, `EXIT_ZOMBIE`, `EXIT_DEAD`状态不会被链接
  > 2、`TASK_INTERRUPTABLE` 或者 `TASK_UNINTERRUPTIBLE`，因为这种状态的进程无法提供充足的信息来快速获取到对应的进程，这类进程会被放在`wait queues`里面

- `wait queue`用途：中断处理，进程同步， timing

- `wait queue`的实现方式是双端链表，并且有`spinlock`来进行同步；链表里面连接的都是等待同一个事件的进程/线程；

  需要注意的是，如果某个事件准备好了，去把所有的进程唤醒往往不是最优的做法，因为很有可能多个进程在等待同一个事件，此时只需要唤醒一个进程就好，其他进程继续sleep

- 旧版本的Linux使用特定机构提供的硬件指令来进行硬件层面的上下文切换。但是新版本不这样做了，原因如下：

  > 1 使用`mov`指令可以更好的检查数据的合法性，可以来检查`ds`和`es segmentation registers`，因为这两个寄存器有可能被恶意用户更改。
  > 2 旧方法和新方法的基本差不多，但是旧方法已经没办法在优化，新方法可以在进行优化。

- `thread_struct`存储了任务的状态，包含平台的很多寄存器信息。如下图所示：

  ```c
  struct thread_struct {
  /* cached TLS descriptors. */
  	struct desc_struct tls_array[GDT_ENTRY_TLS_ENTRIES];
  	unsigned long	esp0;
  	unsigned long	sysenter_cs;
  	unsigned long	eip;
  	unsigned long	esp;
  	unsigned long	fs;
  	unsigned long	gs;
      ......
  }
  ```

- 进程切换主要包含两个步骤：

  > 1 转换Page Global Directory，使其能够安装一个新的地址空间。
  > 2 转换内核栈，硬件上下文，使其能够执行一个新的进程

- 当进程切换的准备工作做好后，就需要调用`switch_to`宏来进行进程切换，该宏的原型为：

  `switch_to(prev, next, last)`

  可以看到，该宏有三个参数，前两个参数可以理解，为什么会有第三个参数呢？原因如下：

  > 假设有三个进程A，B，C，当从A切换到B的时候，prev指向A，next指向B，当从B切换到C的时候，prev指向B，next指向C，如果再从C指向A，**内核发现了A的旧的栈，栈里面存储了prev是A，next是B**，那么C的引用就永远的丢失了；此时第三个参数的目的就是把last赋值给A的prev，避免C引用的丢失。

![](res/switch_to三个参数的原因.PNG)

- `switch_to`最终会调用`__swith_to(prev_p, next_p)`函数，该函数所做的事情如下：

  - 执行`__unlazy_fpu( )` 定义的代码，保存`prev_p`进程的`FPU,MMX,XMM`等寄存器里面的内容

  - 执行`smp_processor_id`( )宏来查找CPU的编号，由该CPU来执行具体的进程

  - 将`next_p->thread.esp0` 读取到上一步指定`CPU TSS`里面的esp0段。

  - 读取`next_p`将要使用到的`Thread-Local Storage`读取到CPU的`Global Descriptor Table`里面，一般有三个tls。

  - 将fs，gs段寄存器里面的内容分别存到`prev_p->thread.fs`和`prev_p->thread.gs`里面

  - 如果prev_p或者next_p使用到了fs或者gs 段寄存器，那么将这些寄存器里面的值存储在next_p的`thread_struct`里面。这部分代码其实很复杂，因为包含了一系列的异常处理，CPU会判断读取到该寄存器里面的值是否正确。

  - 将`next_p->thread.debugreg array`读到`dr0-dr7`这些`debug`寄存器里面，只有当next_p正在使用debug寄存器的时候才会做这些事情。

  - 更新TSS里面的I/O bitmap，只有当 next_p或者prev_p拥有他们自定义的`I/O permission bitmap`的时候才会进行更新。因为进程很少去更新`I/O Permission bitmap`，所以该bitmap只有在进程真正的去访问I/O port的时候，才会进行更新。

  - 代码结束。对应的结束代码：

    ```c
    return prev_p;
    ```

    对应的汇编代码为：

    ```c
    movl %edi,%eax
    ret
    ```

- 通过`clone(), fork(), vfork()` 来创建进程，这些函数最终都会调用到`do_fork()`函数，该函数主要做了如下事情：

  - 从`pidmap_array`里面创建一个新的pid
  - 检查父亲的`ptrace`标志，并确定`debugger`是否也需要追踪创建出来的儿子进程。
  - 调用`copy_address`来拷贝父亲相关的地址空间以及一些关键变量，**该函数做了大量的事情，详情可以查看原书内容**。
  - 检查`CLONE_STOPPED`和`PT_PTRACED`标志，如果有设置，儿子进程也将保持这些标志，直到其他进程对该状态进行了更改。
  - 如果没有设置`CLONE_STOPPED`，那么调用`wake_up_new_task`来对父子进程进行调度。
  - 如果设置了`CLONE_STOPPED`，那么将`child`设置`TASK_STOPPED`状态。
  - 如果父亲进程正在被trace，便将儿子的PID存储在`ptrace_message`里面，并且调用`ptrace_notify()`函数，该函数会停止当前的进程，并且发送`SIGCHLD`信号给其父亲。父亲返回给祖父，也就是debugger，debugger可以通过`current->ptrace_message`获取到儿子的pid，并做进一步的处理。
  - 如果设置了`CLONE_VFORK`，便将父亲进程插入到`wait queue`里面，并且暂时父亲进程直到儿子终止或者运行了个新的程序。
  - 函数结束，并返回儿子的pid。

- 内核线程和普通线程的区别：

  > 1 内核线程只能运行在内核态，而普通线程既可以运行在用户态也可以在内核态。
  >
  > 2 因为内核线程只能运行在内核态，所以其使用linear address只能大于PAGE_OFFSET，而普通线程可以使用所有的liear address。

- 通过`kernel_thread( )`函数可以创建一个内核进程，该函数最终会调用`do_fork`，其调用`do_fork`的方式如下：

  `do_fork(flags|CLONE_VM|CLONE_UNTRACED, 0, pregs, 0, NULL, NULL);`

  `CLONE_VM`可以避免对caller 进程的拷贝，因为内核进程永远不会访问用户态的地址空间。`CLONE_UNTRACED`可以保证内核进程不会被任何进程trace

- 几个kernel thread：

  - `Porcess 0`：所有进程的祖先，也叫`idle process`，历史原因也叫`swapper process`。该进程使用一些静态分配的数据结构，除此之外的其他进程使用的数据结构都是动态申请的。

    当该进程创建完`Process 1`进程（也叫init进程）后，就开始执行`cpu_idle`函数；在多CPU环境下，内核会为每一个CPU都创建一个Process 0进程，当第一个Process 0创建成功后，使用`copy_process`函数来创建其他进程。

  - `Process 1`：也叫`init`进程，一直存在，直到系统关闭。

- `Linux2.6`里面有两个系统调用可以结束进程：
  - `exit_group()`: 终止一个线程组，最终会调用`do_group_exit()`，最终也会调用`do_exit()`来执行真正的退出操作。
  - `_exit()`：结束一个单独的进程，而不管所在进程组里面的其他进程。最终会调用`do_exit()`

- 当进程终止后，kernel不允许立即回收相关的`process descriptor`数据结构，因为有时候父亲需要知道儿子进程的返回值，所以，只有当父亲调用wait()函数的时候，才会进行回收工作，这也是为什么进程状态会有一个`EXIT_ZOMBIE`。如果父亲进程在儿子进程之前终止，那么儿子进程将会变为孤儿进程，这些孤儿进程会被强制设置为`init`进程的儿子，`init`进程通过调用`wait`来结束这些孤儿进程。

- `release_task()`函数负责回收僵尸进程所占有的数据结构，通过两种方式来执行该函数：
  
  - 如果父亲进程对孩子的消息不感兴趣，那么调用`do_exit()`函数来结束。这种方式下，资源回收工作有`scheduler`来进行
  - 当孩子返回信号给父亲，父亲可以通过`wait4()`或者`waitpid()`函数来结束孩子进程，这种方式下，回收工作由上述函数来进行。
  
  关于该函数具体做了什么，可以查看原书在结合代码学习。