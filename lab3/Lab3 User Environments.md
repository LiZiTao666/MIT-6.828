# Introduction
本文主要介绍JOS中的进程，异常处理，系统调用。内容上分为三部分：

1. 用户环境建立，可以加载用户ELF文件并执行。（目前还没有文件系统，需要在内核代码硬编码需要加载的用户程序）
2. 建立异常处理机制，异常发生时能从用户态进入内核进行处理，然后返回用户态。
3. 借助异常处理机制，提供系统调用的能力。

注意：本实验指的用户环境和UNIX中的进程是一个概念！！！！！！！之所有没有使用进程是强调JOS的用户环境和UNIX进程将提供不同的接口。

# Part A: User Environments

    typedef int32_t envid_t; //用户环境ID变量，32位的。

    An environment ID 'envid_t' has three parts:

    +1+---------------21-----------------+--------10--------+
    |0|          Uniqueifier             |   Environment    |
    | |                                  |      Index       |
    +------------------------------------+------------------+
                                          \--- ENVX(eid) --/

    The environment index ENVX(eid) equals the environment's index in the
    'envs[]' array.  The uniqueifier distinguishes environments that were
    created at different times, but share the same environment index.

    All real environments are greater than 0 (so the sign bit is zero).
    envid_ts less than 0 signify errors.  The envid_t == 0 is special, and
    stands for the current environment.

    struct Env {
	    struct Trapframe env_tf;	// Saved registers
	    struct Env *env_link;		// Next free Env
	    envid_t env_id;			// Unique environment identifier
	    envid_t env_parent_id;		// env_id of this env's parent
	    enum EnvType env_type;		// Indicates special system environments
	    unsigned env_status;		// Status of the environment
	    uint32_t env_runs;		// Number of times environment has run

	    // Address space
	    pde_t *env_pgdir;		// Kernel virtual address of page dir
    };
    各个字段解释如下：
    env_tf：Trapframe结构定义在inc/trap.h中，相当于寄存器的一个快照，当前用户环境重新运行时，该结构中保存的寄存器信息将被重新载入到寄存器运行。
    env_link：指向下一个ENV结构，用于构建链表使用。
    env_id：用户环境的id
    env_parent_id：当前用户环境父节点的id
    env_type：对于大部分用户环境是ENV_TYPE_USER，后面将会介绍特殊的系统服务环境
    env_status：当前用户环境状态
    env_pgdir：页目录地址

就像Unix中的进程一样，一个JOS环境中结合了“线程”和“地址空间”的概念。线程通常是由被保存的寄存器的值来定义的，而地址空间则是由env_pgdir所指向的页目录表还有页表来定义的。为了运行一个用户环境，内核必须设置合适的寄存器的值以及合适的地址空间。

我们的struct Env类似于xv 6中的struct proc。这两种结构都在TrapFrame结构中保存环境的(即进程的)用户模式寄存器的状态。在Jos中，单个环境不像xv 6中的进程那样有自己的内核堆栈。一次只能在内核中激活一个Jos环境，因此Jos只需要一个内核堆栈。

注意的实现细节：每次生成的env_id的第11至第31位（即Uniqueifier）都比上次多1，由此可以区分所有不同的共享某块物理内存的所有进程，实现细节如下

    // Generate an env_id for this environment.
    generation = (e->env_id + (1 << ENVGENSHIFT)) & ~(NENV - 1);
    if (generation <= 0)    // Don't create a negative env_id.
        generation = 1 << ENVGENSHIFT;
    e->env_id = generation | (e - envs);
    
![](1.png)

---

## Creating and Running Environments

本节需要实现env相关函数，以下是实现过程的一些要用到的知识。

---

## GDT&&LDT
现在您将用Kern/env.c编写运行用户环境所必需的代码。因为我们还没有文件系统，所以我们将设置内核来加载嵌入在内核中的静态二进制镜像文件。Jos将此二进制文件作为ELF可执行镜像文件嵌入内核中。

首先，再补充一些关于GDT和LDT的知识

![](2.png)

LDT和GDT从本质上说是相同的，只是LDT嵌套在GDT之中。LDTR记录局部描述符表的起始位置，与GDTR不同LDTR的内容是一个段选择子。由于LDT本身同样是一段内存，也是一个段，所以它也有个描述符描述它，这个描述符就存储在GDT中，对应这个表述符也会有一个选择子，LDTR装载的就是这样一个选择子。LDTR可以在程序中随时改变，通过使用lldt指令。如上图，如果装载的是Selector 2则LDTR指向的是表LDT2。

其实只要知道LDT就是GDT中的一些段。然后我们有LDTR来指向LDT的起始地址，所以LDTR里面装的是段选择子

下面具体分析一下GDT，它具体的结构如下

![](3.png)

![](4.png)

在代码中表现形式如下

    // Segment Descriptors
    struct Segdesc {
	    unsigned sd_lim_15_0 : 16;  // Low bits of segment limit
	    unsigned sd_base_15_0 : 16; // Low bits of segment base address
	    unsigned sd_base_23_16 : 8; // Middle bits of segment base address
	    unsigned sd_type : 4;       // Segment type (see STS_ constants)
	    unsigned sd_s : 1;          // 0 = system, 1 = application
	    unsigned sd_dpl : 2;        // Descriptor Privilege Level
	    unsigned sd_p : 1;          // Present
	    unsigned sd_lim_19_16 : 4;  // High bits of segment limit
	    unsigned sd_avl : 1;        // Unused (available for software use)
	    unsigned sd_rsv1 : 1;       // Reserved
	    unsigned sd_db : 1;         // 0 = 16-bit segment, 1 = 32-bit segment
	    unsigned sd_g : 1;          // Granularity: limit scaled by 4K when set
	    unsigned sd_base_31_24 : 8; // High bits of segment base address
    };

具体细节可参考[这篇文章](https://blog.csdn.net/abc123lzf/article/details/109289567)

---

## ELF

我们需要加载用户ELF文件并执行。（因为目前还没有文件系统，所以需要在内核代码硬编码需要加载的用户程序），因此我们要实现一个类似ELF可执行文件加载器的功能。

ELF格式可参考[这篇文章](https://www.cnblogs.com/gatsby123/p/9750187.html)和CSAPP第七章。

---

## 总结
用户进程的代码被调用前，操作系统一共按顺序执行了以下几个重要函数：

1. start (kern/entry.S)
2. i386_init (kern/init.c)
3. cons_init（控制台初始化）
4. mem_init（给内核页目录和页表结构分配空间并且建立映射，给进程结构分配空间）
5. env_init（初始化所有进程结构，建立进程链表）
6. trap_init（目前还未实现）
7. env_create（创建进程）
8. env_run（运行进程，调用env_pop_tf函数）

env_create内部函数调用

    env_create()
        -->env_alloc()
            -->env_setup_vm()
        -->load_icode()
            -->region_alloc()

一旦你完成上述函数（除了trap_init）的代码，并且在QEMU下编译运行，系统会进入用户空间，并且开始执行hello程序，直到它做出一个系统调用指令int。但是这个系统调用指令不能成功运行，因为到目前为止，JOS还没有设置相关硬件来实现从用户态向内核态的转换功能。当CPU发现，它没有被设置成能够处理这种系统调用中断时，它会触发一个保护异常，然后发现这个保护异常也无法处理，从而又产生一个错误异常，然后又发现仍旧无法解决问题，所以最后放弃，我们把这个叫做"triple fault"。通常来说，接下来CPU会复位，系统会重启。

# Part B: Handling Interrupts and Exceptions

注意：在这个实验中，我们通常引用Intel关于中断、异常等的术语。但是，异常、陷入、中断、出错和终止（exception, trap, interrupt, fault and abort）等术语在体系结构或操作系统之间没有标准意义，而且在特定体系结构(如x86)上使用时往往忽略它们之间的细微区别。当你在实验之外看到这些术语时，其含义可能会略有不同。

## Basics of Protected Control Transfer


异常(Exception)和中断(Interrupts)都是“受到保护的控制转移方法”，都会使处理器从用户态转移为内核态（CPL = 0），而不给用户模式代码任何机会来干扰内核或其他环境的运行。在Intel的术语中，中断指的是由外部异步事件引起的处理器控制权转移，比如外部IO设备发送来的中断信号。相反，异常则是由于当前正在运行的指令所带来的同步的处理器控制权的转移，比如访问无效内存，或者除零溢出。

为了确保这些控制转移能够真正被保护起来，需要设计处理器的中断/异常机制，使当前在中断或异常发生时运行的代码不能任意选择内核输入的位置或方式。在x86上，有两种机制配合工作来提供这种保护：

1. 中断向量表 (interrupt descriptor table) (缩写为IDT) ：处理器保证中断和异常只能够引起内核进入到一些特定的，被事先定义好的程序入口点，而不是由触发中断的程序来决定中断程序入口点。
   
   X86允许多达256个不同的中断或异常，每一个都配备一个独一无二的中断向量。一个向量指的就是0到255中的一个数。一个中断向量的值是根据中断源来决定的：不同设备，错误条件，以及对内核的请求都会产生出不同的中断和中断向量的组合。CPU将使用这个向量作为这个中断在处理器中断向量表中的索引，这个表是由内核设置的，放在内核空间中，和GDT很像。通过这个表中的任意一个表项，处理器可以知道：

   * 需要加载到EIP寄存器中的值，这个值指向了处理这个中断的中断处理程序的位置。
   * 需要加载到CS寄存器中的值，里面还包含了这个中断处理程序的运行特权级。（即这个程序是在用户态还是内核态下运行。）

2. 任务状态段 (The Task State Segment) (缩写为TSS) ：处理器需要在中断或异常发生之前保存旧的处理器状态，例如在处理器调用异常处理程序之前EIP和CS的值。以便异常处理程序可以在以后恢复旧状态并恢复中断的代码。但是，必须保护旧处理器状态的保存区域不受非特权用户模式代码的影响；否则，错误或恶意的用户代码可能会危及内核。
   
   因此，当x86处理器接受中断或陷阱，导致特权级别从用户模式更改到内核模式时，它还会切换到内核内存中的堆栈。称为TSS的结构指定了该堆栈所在的段选择器和地址。处理器压入(在这个新堆栈上)SS、ESP、EFLAGS、CS、EIP和一个可选的错误代码。然后从IDT加载CS和EIP，并将ESP和SS设置为新堆栈的地址。

   虽然TSS很大，并且可能有多种用途，但Jos只使用它来定义处理器在从用户模式转换到内核模式时应该切换到的内核堆栈。由于Jos中的“内核模式”在x86上的特权级别为0，处理器只使用TSS的ESP0和SS0字段来定义内核堆栈，而不使用其它字段。

到目前我们已经碰到很多除通用寄存器之外的寄存器了，下图总结了各种寄存器：

![](5.png)

1. TSS选择器就是刚才用ltr指令设置的。中断发生时，自动通过该寄存器找到TSS结构（JOS中是ts这个变量），将栈寄存器SS和ESP分别设置为其中的SS0和ESP0两个字段的值，这样栈就切换到了内核栈。
2. GDTR就是全局描述符表寄存器，之前已经设置过了。
3. PDBR是页目录基址寄存器，通过该寄存器找到页目录和页表，将虚拟地址映射为物理地址。
4. IDTR是中断描述符表寄存器，通过这个寄存器中的值可以找到中断表。

---

## Types of Exceptions and Interrupts

X86处理器内部生成的所有同步异常都使用0到31之间的中断向量，因此映射到IDT项0-31。例如，缺页中断就是14号。大于31的中断向量使用于软件中断，软件中断可以由int指令生成，或者在外部设备需要注意时由外部设备引起异步硬件中断。

在本节中，我们将扩展Jos以处理内部生成的x86异常向量0-31。在下一节中，我们将使用Jos处理软件中断向量48(0x30)，Jos(相当任意)使用其作为系统调用中断向量。在实验4中，我们将扩展Jos以处理外部生成的硬件中断，例如时钟中断。

---

## An Example

假设处理器在用户环境中执行代码，并遇到试图除以零的除法指令。

1. 处理器切换到TSS的SS0和ESP0字段定义的堆栈，这两个字段在Jos中将分别保存GD_KD和KSTACKTOP值。
2. 处理器将异常参数压入到内核堆栈上，从地址KSTACKTOP开始：

                    +--------------------+ KSTACKTOP             
                    | 0x00000 | old SS   |     " - 4
                    |      old ESP       |     " - 8
                    |     old EFLAGS     |     " - 12
                    | 0x00000 | old CS   |     " - 16
                    |      old EIP       |     " - 20 <---- ESP 
                    +--------------------+             

3. 除以0的异常中断号是0，处理器读取IDT的第0项，并设置CS:EIP。
4. 异常处理函数控制并处理这个异常，通过例如终止这个用户环境等方式。

对于某些类型的x86异常，除了压入上图“标准”的5个字之外，处理器还将包含错误码的另一个字压入堆栈。请参阅80386手册，以确定处理器推送错误码的异常号，以及错误码在这种情况下意味着什么。堆栈在异常处理程序的开头如下所示：

                    +--------------------+ KSTACKTOP             
                    | 0x00000 | old SS   |     " - 4
                    |      old ESP       |     " - 8
                    |     old EFLAGS     |     " - 12
                    | 0x00000 | old CS   |     " - 16
                    |      old EIP       |     " - 20
                    |     error code     |     " - 24 <---- ESP
                    +--------------------+             

---

## Nested Exceptions and Interrupts (Nested意为嵌套)

处理器在用户态下和内核态下都可以处理异常和中断。但是，只有当从用户模式进入内核模式时，x86处理器才会自动切换堆栈，然后将其旧的寄存器状态压入堆栈，并通过IDT调用相应的异常处理函数。如果当中断或异常发生时，处理器已经处于内核模式(CS寄存器的低2位已经为零)，那么CPU只需在同一个内核堆栈上压入更多的值。这样，内核就可以优雅地处理由内核内部代码引起的嵌套异常。此功能是实现保护的重要工具，我们将在后面关于系统调用的部分中看到这一点。

如果处理器已经处于内核模式，并遇到嵌套异常，因为它不需要切换堆栈，所以它不会保存旧的SS或ESP寄存器。如果这个异常类型不压入错误码，内核堆栈在进入异常处理程序时如下所示：

                    +--------------------+ <---- old ESP
                    |     old EFLAGS     |     " - 4
                    | 0x00000 | old CS   |     " - 8
                    |      old EIP       |     " - 12
                    +--------------------+             

对于需要压入错误码的异常，处理器如前所述在压入旧EIP之后立即压入错误码。

警告：如果处理器处于内核模式时遇到异常，并且由于缺乏堆栈空间等原因无法将其旧状态信息（比如寄存器值）压入内核堆栈，那么处理器就无法恢复原来状态，它只能自动重启。不用说，内核的设计应该使这种情况不会发生。

---

## Setting Up the IDT

[如果想知道各个中断具体是什么，可以看这个。](https://pdos.csail.mit.edu/6.828/2018/readings/i386/s09_08.htm)

![](6.png)

[如果想知道堆栈切换具体流程，可以看这个。](https://blog.csdn.net/cinmyheart/article/details/45270559)

你要实现的代码的效果如下： 
          
            IDT             trapentry.S                 trap.c

    +----------------+                        
    |   &handler1    |-----> handler1:          trap (struct Trapframe *tf)
    |                |      // do stuff         {
    |                |      call trap           //handle the exception/interrupt
    |                |      // ...              }
    +----------------+
    |   &handler2    |-----> handler2:
    |                |      // do stuff
    |                |      call trap
    |                |      // ...
    +----------------+
            .
            .
            .
    +----------------+
    |   &handlerX    |-----> handlerX:
    |                |      // do stuff
    |                |      call trap
    |                |      // ...
    +----------------+

每一个中断或异常都有它自己的中断处理函数，分别定义在 trapentry.S中，trap_init()将初始化IDT表。每一个处理函数都应该构建一个结构体 Trapframe 在堆栈上，并且调用trap()函数且其参数为这个结构体，trap()然后处理异常/中断，或分派到特定的处理函数。

所以整个操作系统的中断控制流程为：

1. trap_init() 先将所有中断处理函数的起始地址放到中断向量表IDT中。
2. 当中断发生时，不管是外部中断还是内部中断，处理器捕捉到该中断，进入内核态，根据中断向量去查询中断向量表，找到对应的表项
3. 保存被中断的程序的上下文到内核堆栈中，调用这个表项中指明的中断处理函数。
4. 执行中断处理函数。
5. 执行完成后，恢复被中断的进程的上下文，返回用户态，继续运行这个进程。



























