# 分类
从不同角度分类：
根据中断来源：内部中断、外部中断
根据中断是否可屏蔽：可屏蔽中断、不可屏蔽中断
根据终端入口跳转方法：向量中断、非向量中断，向量中断由硬件提供中断服务程序入口地址，非向量中断由软件提供中断服务程序入口地址，进入中断函数之后再通过软件中断标志判断具体是哪个中断

同步中断：来自cpu内部，即软中断，当指令执行时由cpu控制单元产生，之所以称为同步是因为只有在一条指令终止执行后cpu才会发出中断
异步中断：来自cpu外部，即硬件中断，由其他硬件设备依照cpu时钟信号随机产生的

intel处理手册中把同步和异步中断分被称为异常（exception）和中断（interrupt）

异常又分为故障(Fault) 和陷阱(Trap)，它们的共同特点是既不使用中断控制器，又不能被屏蔽（异常其实是CPU发出的中断信号）。
陷阱（trap）：它通常是一种由异常情况（例如，断点、被零除、无效内存访问）引起的同步中断。
故障（Fault）：
    Fault exception is used in a client application to catch contractually-specified SOAP faults. By the simple exception message, you can’t identify the reason of the exception, that’s why a Fault Exception is useful.
    

```c
x86

中断 --+-- 同步中断(异常、软件中断) --+-- 故障
       |                              |
       |                              +-- 陷阱
       |                              |
       |                              +-- 异常终止
       |                              |
       |                              +-- 编程异常
       |
       +-- 异步中断(中断、硬件中断) --+-- 可屏蔽中断
                                      |
                                      +-- 不可屏蔽中断
                            
从0~31的向量对应于异常和不可屏蔽中断。
从32~47的向量（即由I/O设备引起的中断）分配给可屏蔽中断。
剩余的从48~255的向量用来标识软中断。Linux只用了其中的一个（即128或0x80向量）用来实现系统调用。

说明，可以在proc文件系统下的interrupts文件中，查看当前系统中各种外设的IRQ：
$cat /proc/interrupts
```

- 可屏蔽中断
x86对于外部的中断控制器通常使用 8259A，具体使用细节参考 《微型计算机原理及应用》

对于外部I/O请求的屏蔽可分为两种情况：
一种是从CPU的角度，也就是清除eflag的中断标志位（IF），当IF=0时，禁止任何外部I／O的中断请求，即关中断；一种是从中断控制器的角度，因为中断控制器中有一个8位的中断屏蔽寄存器，每位对应8259A中的一条中断线，如果要禁用某条中断线，则把中断屏蔽寄存器相应的位置1，要启用，则置0。

- 异常和不可屏蔽中断
在CPU执行一个异常处理程序时，就不再为其他异常或可屏蔽中断请求服务，也就是说，当某个异常被响应后，CPU清除eflag的中IF位，禁止任何可屏蔽中断。但如果又有异常产生，则由CPU锁存（CPU具有缓冲异常的能力），待这个异常处理完后，才响应被锁存的异常。我们这里讨论的异常中断向量在0～31之间，不包括系统调用（中断向量为0x80）。

Intel x86处理器发布了大约20种异常（具体数字与处理器模式有关）。Linux内核必须为每种异常提供一个专门的异常处理程序。


嵌入式系统及 x86 PC中大多包含可编程中断控制器（PIC），许多MCU内部就集成了PIC。如80386中，PIC是两片 i8259A 芯片的级联。在ARM多核处理器里最常用的中断控制器是 GIC （Generic Interrupt Controller）。
关于 SGI（software generated interrupt）、PPI（private peripheral interrupt）、SPI（shared peripheral interrupt）参考 《Linux 设备驱动开发详解》（宋宝华）

# Linux中断机制

Linux中断机制由三部分组成：
1、中断子系统初始化：内核自身初始化过程中对中断处理机制初始化，例如中断的数据结构以及中断请求等。
2、中断或异常处理：中断整体处理过程。
3、中断API：为设备驱动提供API，例如注册，释放和激活等。

## 中断子系统初始化
### 中断描述符表（IDT）初始化

中断描述符表初始化需要经过两个过程：
第一个过程在内核引导过程。由两个步骤组成，首先给分配IDT分配2KB空间（256中断向量，每个向量由8bit组成）并初始化；然后把IDT起始地址存储到 IDTR 寄存器中。
第二个过程内核在初始化自身的start\_kernal函数中使用trap\_init初始化系统保留中断向量，使用init_IRQ完成其余中断向量初始化。

### 中断请求队列初始化

init\_IRQ调用pre\_intr\_init\_hook,进而最终调用init\_ISA\_irqs初始化中断控制器以及每个IRQ线的中断请求队列。

## 中断或异常处理

中断处理过程：设备产生中断，并通过中断线将中断信号送往中断控制器，如果中断没有被屏蔽则会到达CPU的INTR引脚，CPU立即停止当前工作，根据获得中断向量号从IDT中找出门描述符，并执行相关中断程序。

异常处理过程：异常是由CPU内部发生所以不会通过中断控制器，CPU直接根据中断向量号从IDT中找出门描述符，并执行相关中断程序。

中断控制器处理主要有5个步骤：
1. 中断请求
2. 中断相应
3. 优先级比较
4. 提交中断向量
5. 中断结束
这里不再赘述5个步骤的具体流程。

CPU处理流程主要有6个步骤：
1. 确定中断或异常的中断向量
2. 通过IDTR寄存器找到IDT
3. 特权检查
4. 特权级发生变化，进行堆栈切换
5. 如果是异常将异常代码压入堆栈，如果是中断则关闭可屏蔽中断
6. 进入中断或异常服务程序执行
这里不再赘述6个步骤的具体流程。

## /proc 接口
当硬件的中断达到处理器时，一个内部计数递增，这为检查设备是否按预期工作提供了一种方法，产生的中断报告显示在文件 /proc/interrupts 中。
文件 /proc/interrupt 给出了已经发到系统上每一个 CPU 的中断数量：
- 列是中断号，该文件只会显示已经注册了中断处理程序的中断号
- 中间的数字表示当前中断产生的次数
- 最后两列表示中断控制器和注册了中断处理程序的设备名称

/proc/stat 记录了一些系统活动的底层统计信息，包括（但不限于）从系统启动开始接收到的中断数量，stat文件的每行都以一个字符串开始，他是这行的关键字。例如 intr 即表示中断，例如：
intr 4013566 9 0 0 0 2 0 0 0 1 0 0 0 0 0 0 0 6 0 2803 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0  0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 447183 21878 38404 0 246696 42 30 0 0 0 0 11 1534 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0  0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0  
第一个数字是所有中断的总数，而其他的每个数字都代表一个单独的IRQ信号线，从中断0开始。

stat 和 interrupts 两个文件的不同之处是 interrupts 文件不依赖于体系结构，而 stat 文件则是依赖的：字段的数量依赖于内核之下的硬件。

## 中断API

内核提供的API主要用于驱动的开发。
```c
==> 注册IRQ：
    int request_irq(unsigned int irq,         // 中断号
                    irq_handler_t handler,    // 中断处理程序 typedef   irqreturn_t (*irq_handler_t)(int, void *);  注意其中的返回值 irqreturn_t  （typedef   enum irqreturn irqreturn_t;）
                    unsigned long flags,
                    const char *name,         // 与中断相关的设备的ASCII文本表示。例如，PC机上键盘中断对应的这个值为“keyboard”。这些名字会被 /proc/irq 和 /proc/interrupts 文件使用，以便与用户通信
                    void *dev);               // 在共享中断线的时候使用，当一个中断处理程序需要释放时，dev提供唯一的标志信息（cookie），以便从共享的诸多中断处理程序中删除指定的那一个，建议每次调用都传递这个参数

关于 flags，可以设置为0,也可以是一个或者多个标志的掩码。其定义在文件<linux/interrupt.h> 中，其中比较重要的是：
IRQF_DISABLED：
该标志被设置之后，意味着内核在处理中断处理程序期间，要禁止其他的所有中断。如果不设置，中断处理程序可以与除本身外的其他任何中断同时运行。多数中断处理程序是不会去设置该位的
IRQF_SAMPLE_RANDOM：
表明这个设备产生的中断对内核熵池有贡献。内核熵池负责提供从各种随机事件导出的真正随机数。如果指定了该标志，那么来自该设备的中断间隔时间就会作为熵填充到熵池
IRQF_TIMER：
该标志是特别为系统定时器的中断处理而准备的
IRQF_SHARED：
该标志表示可以在多个中断处理程序之间共享中断线。在同一个给定的线上注册的每个处理程序必须指定这个标志，否则每条线上只能有一个处理程序
返回参数：
通常返回0时表示申请成功，为负值时表示错误码。返回-EBUSY表示已经有另一个驱动程序占用了要请求的中断信号线

中断处理程序
static irqreturn_t intr_handler(int irq, void *dev)
其中：
irq:这个处理程序要响应的中断的中断号，如今已经很少用了，只是在打印log时可能会用，在之前的kernel里，是没有dev参数的，因此必须使用中断号来区分使用相同驱动的程序
irqreturn_t：中断处理程序可能返回两个特殊的值，IRQ_NONE 和 IRQ_HANDLED。当中断处理程序检测到一个中断，但是该中断对应的设备并不是在注册处理函数期间指定产生源时，返回 IRQ_NONE；当中断处理程序被正确调用，并且确实是他所对应的设备产生了中断时，返回 IRQ_HANDLED。另外也可以使用宏 IRA_RETVAL(val) 返回，IRA_RETVAL(0) 即 IRQ_HANDLED，否则为 IRQ_NONE

注意，request_irq函数可能会睡眠，所以不能在中断上下文或者其他不允许阻塞的代码中调用该函数。睡眠是因为在注册的过程中内核需要在 /proc/irq 文件中创建一个与中断对应的项。函数proc_mkdir()就是用来创建这个新的procfs项的。proc_mkdir() 通过调用函数 proc_create() 对这个新的 profs 进行设置，而 proc_create() 会调用函数 kmalloc() 来请求分配内存，kmalloc 是可以睡眠的

static   inline   int __must_check   devm_request_irq(struct device *dev,
                                                      unsigned   int irq,
                                                      irq_handler_t handler,
                                                      unsigned   long irqflags,
                                                      const   char *devname,
                                                      void *dev_id)
此函数与 request_irq() 的区别是 devm_ 开头的API申请的是内核 “managed” 的资源，一般不需要在出错处理和 remove() 接口里再显式的释放

==> 释放IRQ：
    void free_irq(unsigned int, void *); // 如果中断不共享，该函数删除处理程序的同时将禁用这条中断线。如果中断是共享的，则仅删除dev对应的处理程序，而这条中断线在删除了最后一个处理程序时才会被禁用

注：IRQ线资源非常宝贵，我们在使用时必须先注册，不使用时必须释放IRQ资源。

==> 激活当前CPU中断：
    local_irq_enable()；

==> 禁止当前CPU中断：
    local_irq_disable();

==> 禁止当前CPU中断，保存当前状态：
    local_irq_save(flags);

==> 使能当前CPU中断，恢复之前的状态：
    local_irq_restore(flags);

==> 激活指定中断线，注意enable和disable的数量必须匹配，即如果disable了两次，必须enable两次，不然无法使能：
    void enable_irq(unsigned int irq);

==> 禁止指定中断线，只有在当前执行的所有处理程序完成后才能返回：
    void disable_irq(unsigned int irq);

==> 禁止指定中断线，不会等待当前中断处理程序执行完毕：
    void disable_irq_nosync(unsigned int irq);

==> 等待一个特定中断处理程序退出
    void synchronize_irq(unsigned int irq);

==> 检查内核是否处于中断上下文中，无论上半部分还是下半部分，没有在处理中断返回0：
    in_interrupt()
    
==> 内和确实正在执行中断处理程序时返回非0：
    in_irq()

==> 使能当前处理器的软中断和tasklet，同样 disable 的次数要和 enable 的次数对应
    void locak_bh_enable()
    
==> 禁止当前处理器的软中断和tasklet
    void locak_bh_disable()

注：此函数调用irq_chip中disable禁止指定中断线，所以不会保证中断线上执行的中断服务程序已经退出。
```

## 关于共享中断
- request_irq() 的参数 flags 必须设置 IRQF_SHARED 标志
- dev参数不能为 NULL
- 终端处理程序必须能够区分他的设备是否真的产生了中断

内核接收到一个中断后，它将依次调用在该中断线上注册的没一个处理程序。因此，一个处理程序必须知道他是否应该为这个中断负责。如果与他相关的设备并没有产生中断，那么处理程序应该立即退出。这需要硬件设备提供状态寄存器（或类似机制），以便中断处理程序进行检查。然而大多数硬件都能够提供这种功能。

## 关于中断栈
曾经，中断处理程序并不具有自己的栈。他们共享所中断进程的内核栈。内核栈的大小是两页，具体地说，在32位体系结构上是8KB，在64位体系结构上是16KB。
在2.6版早期的内核中，增加了一个选项，把栈的大小从两页减到一页，也就是在32位的系统上只提供4KB的栈。这就减轻了内存的压力，为了应对栈空间的减少，中断处理程序拥有了自己的栈，每个处理器一个，大小为一页，这个栈就称为中断栈。


## 中断机制划分
由于中断会打断内核中进程的正常调度运行，所以要求中断服务程序尽可能的短小精悍；但是在实际系统中，当中断到来时，要完成工作往往进行大量的耗时处理。因此期望让中断处理程序运行得快，并想让它完成的工作量多，这两个目标相互制约，诞生——顶/底半部机制。

中断处理程序是顶半部——接受中断，它就立即开始执行，但只有做严格时限的工作。能够被允许稍后完成的工作会推迟到底半部去，此后，在合适的时机，底半部会被开终端执行。顶半部简单快速，执行时禁止一些或者全部中断。

底半部稍后执行，而且执行期间可以响应所有的中断。这种设计可以使系统处于中断屏蔽状态的时间尽可能的短，以此来提高系统的响应能力。顶半部只有中断处理程序机制，而底半部的实现有软中断，tasklet和工作队列实现。

### 顶/底半部划分原则
　1） 如果一个任务对时间非常敏感，将其放在顶半部中执行；
　2） 如果一个任务和硬件有关，将其放在顶半部中执行；
　3） 如果一个任务要保证不被其他中断打断，将其放在顶半部中执行；
　4） 其他所有任务，考虑放置在底半部执行。

### 底半部实现
#### 软中断
##### 机制
软中断作为下半部机制的代表，是随着SMP（share memory processor）的出现应运而生的，它也是tasklet实现的基础（tasklet实际上只是在软中断的基础上添加了一定的机制）。软中断一般是“可延迟函数”的总称，有时候也包括了tasklet（请读者在遇到的时候根据上下文推断是否包含tasklet）。它的出现就是因为要满足上面所提出的上半部和下半部的区别，使得对时间不敏感的任务延后执行，软中断执行中断处理程序留给它去完成的剩余任务，而且可以在多个CPU上并行执行，使得总的系统效率可以更高。它的特性包括：
a）产生后并不是马上可以执行，必须要等待内核的调度才能执行。
b）软中断不能被自己打断，只能被硬件中断打断（上半部）。
c）可以并发运行在多个CPU上（即使同一类型的也可以）。所以软中断必须设计为可重入的函数（允许多个CPU同时操作），因此也需要使用自旋锁来保护其数据结构。

软中断是在编译期间静态分配的。软中断由 softirq_action 结构表示，它定义在 \<linux/interrupt.h\> 中：
```c
structr softirq_action {
    void (*action)(struct softirq_action *);
}
```
kernel/softirq.c 中定义了一个包含有32个该结构体的数组：
- static struct softirq_action softirq_vec[NR_SOFTIRQS];
每个被注册的软中断都占据该数组的一项，因此最多可能有32个软中断，这是一个定值。内核中定义了几种软中断的用途：
```c
enum
{
   HI_SOFTIRQ=0,             // 优先级高的 tasklet
   TIMER_SOFTIRQ,            // 定时器的下半部
   NET_TX_SOFTIRQ,           // 发送网络数据包
   NET_RX_SOFTIRQ,           // 接收网络数据包
   BLOCK_SOFTIRQ,            // BLOCK 装置
   BLOCK_IOPOLL_SOFTIRQ,
   TASKLET_SOFTIRQ,          // 正常优先级的 tasklets
   SCHED_SOFTIRQ,            // 调度程序
   HRTIMER_SOFTIRQ,          // 高分辨率定时器
   RCU_SOFTIRQ,              // RCU 锁定
   NR_SOFTIRQS
};
```

一个软中断必须要被标记之后才会执行。这被称作触发软中断(raising the softirq)。通常，中断处理程序会在返回前标记它的软中断，使其在稍后被执行。在如下的位置，软中断会被检查和执行：
- 从一个硬件中断代码处返回时
- 在 ksoftirqd 内核线程中
- 在那些显式检查和执行待处理的软中断的代码中，如网络子系统中
无论用什么方法唤醒，软中断都要在 do_softirq() 中执行。如果有待处理的软中断。do_softirq() 会循环遍历每一个，调用他们的处理程序。

目前只有两个子系统 (网络和SCSI) 直接使用软中断。此外，内核定时器和 tasklet 都是建立在软中断上的。如果你想加入一个新的软中断，首先应该问问自己为什么用 tasket 实现不了。

##### 使用
添加一个新的软中断首先要在枚举类型中添加一个新的枚举

在运行时通过调用 open_softirq() 注册软中断处理程序，该函数有两个参数：软中断的索引号和处理函数。例如：
open_softirq(NET_TX_SOFTIRQ, net_tx_action);
open_softirq(NET_RX_SOFTIRQ, net_rx_action);
软中断处理程序执行的时候，允许响应中断，但他自己不能休眠。在一个处理程序运行的时候，当前处理器上的软中断被禁止。但其他的处理器仍然可以执行别的软中断。实际上，如果同一个软中断在它被执行的同时再次被触发了，那么另外一个处理器可以同时运行其处理程序。这意味着任何共享数据都需要严格的锁保护。大部分软中断处理程序都通过采取单处理器数据（仅属于某一个处理器的数据，因此根本不需要加锁）或其他一些技巧来避免显式的加锁，从而提供更出色的性能。

在通过枚举类型的列表中添加新项以及调用 open_softirq() 进行注册以后，新的软中断处理程序就能运行。raise_softirq() 函数可以将一个软中断设置为触发状态，让他在下次调用 do_softirq() 函数时投入运行。例如：
raise_softirq(NET_TX_SOFTIRQ);
该函数在触发一个软中断之前要先禁止中断，触发后再恢复原来的状态。如果中断本来就已经被禁止了，那么可以调用另一个函数 raise_softirq_irqoff() ，这会带来一些优化效果。如：
/*
 * 中断已经被禁止
 \*/
raise_softirq_irqoff(NET_TX_SOFTIRQ);
在中断处理程序中触发软中断是最常见的形式。中断处理程序执行硬件设备相关的操作，然后触发相应的软中断，最后退出。内核在执行完中断处理程序之后，马上就会调用 do_softirq() 函数。于是软中断开始执行中断处理程序的剩余任务。

#### Tasklet
##### 机制
tasklet 是一种基于软中断实现的灵活性强、动态创建的下半部实现机制，所以它本身也是软中断，另外tasklet可以通过代码进行动态注册。软中断用轮询的方式处理。假如正好是最后一种中断，则必须循环完所有的中断类型，才能最终执行对应的处理函数。显然当年开发人员为了保证轮询的效率，于是限制中断个数为32个。为了提高中断处理数量，顺道改进处理效率，于是产生了tasklet机制。tasklet 由两类软中断代表：HI_SOFTIRQ 和 TASKLET_SOFTIRQ 。这两者之间唯一的实际区别在于， HI_SOFTIRQ 类型的软中断先于 TASKLET_SOFTIRQ 类型的软中断执行。

Tasklet采用无差别的队列机制，有中断时才执行，免去了循环查表之苦。Tasklet作为一种新机制，显然有更多的优点。正好这时候SMP越来越火了，因此又在tasklet中加入了SMP机制，保证同种中断只能在一个cpu上执行。在软中断时代，显然没有这种考虑。因此同一种软中断可以在两个cpu上同时执行，很可能造成冲突。

总结下tasklet的优点：
- 无类型数量限制；
- 效率高，无需循环查表；
- 支持SMP机制；
它的特性如下：
- 一种相同类型的tasklet只能运行在一个CPU上，不能并行，只能串行执行。
- 多个不同类型的tasklet可以并行在多个CPU上。
- 软中断是静态分配的，在内核编译好之后，就不能改变。但tasklet就灵活许多，可以在运行时改变（比如添加模块时）。

tasklet 由 tasklet_struct  结构表示。每个结构体单独代表一个 tasklet，它在 linux/interrupt.h 中定义为：
```c
struct   tasklet_struct     
{
    struct   tasklet_struct *next;     // 链表中的下一个 tasklet
    unsigned long state;               // tasklet 的状态，只能在 0、TASKLET_STATE_SCHED 和 TASKLET_STATE_RUN 之间选择。 
                                       //   TASKLET_STATE_SCHED 表示 tasklet 已经被调度，正准备投入运行
                                       //   TASKLET_STATE_RUN 表示正在运行，这只有在多处理器的系统上才会作为一种优化来使用，单核处理器系统任何时候都清楚单个 tasklet 是否正在运行
    atomic_t count;                    // 引用计数器，如果不为 0 则 tasklet 被禁止，不允许执行；只有当它为 0 时，tasklet 才被激活，并且在被设置为挂起状态时，该tasklet才能够执行。
    bool use_callback;                 // tasklet 处理函数
    union {
        void (*func)(unsigned   long data);   
        void (*callback)(struct   tasklet_struct *t);   
    };
    unsigned   long data;              // 给 tasklet 处理函数的参数
};
```

已调度的 tasklet （等同于被触发的软中断）存放在两个单处理器数据结构：tasklet_vec (普通 tasklet) 和 tasklet_hi_vec (高优先级的 tasklet)。这两个数据结构都是由 tasklet_struct 结构体构成的链表。tasklet 由 tasklet_schedule() 和 tasklet_hi_schedule() 函数进行调度，他们接受一个指向 tasklet_struct 结构的指针作为参数。
tasklet_schedule() 的执行步骤：
1. 检查 tasklet 的状态是否为 TASKLET_STATE_SCHED。如果是，说明 tasklet 已经被调度过了，函数立即返回
2. 调用 \_tasklet_schedule()。
3. 保存中断状态，然后禁止本地中断。在我们执行 tasklet 代码时，这么做能够保证当 tasklet_schedule() 处理这些 tasklet 时，处理器上的数据不会弄乱。
4. 把需要调度的 tasklet 加到每个处理器一个的 tasklet_vec 链表 或 tasklet_hi_vec 链表的表头上。
5. 唤起 TASKLET_SOFTIRQ 或 HI_SOFTIRQ 软中断，这样在下一次调用 do_softirq() 时就会执行该 tasklet
6. 恢复中断到原状态并返回

在 do_softirq() 中会执行相应的软中断处理程序，而 tasklet_action() 和 tasklet_hi_action() 就是 tasklet 处理的核心，执行步骤可以参考 《Linux 内核设计与实现》P116

##### 使用
静态创建（有一个它的直接引用）：
DECLARE_TASKLET(name, \_callback)
DECLARE_TASKLET_DISABLED(name, \_callback)
这两个宏都是根据给定的名称静态创建一个 tasklet_struct 结构。当该 taslet 被调度以后，给定的函数 func 会被执行。这两个宏的区别在于引用计数器的初始值设置不同。第一个设置为 0 ，表示当前 tasklet 处于激活的状态，第二个设置为 1，表示未激活。

动态创建（间接引用）：
tasklet_init(t, tasklet_handler, dev);

编写 tasklet 处理函数
原型：
void tasklet_handler(unsigned long data)
因为是靠软中断实现，所以 tasklet 不能睡眠。这意味着不能在 tasklet 中使用信号量或者其他阻塞式的函数。由于 tasklet 运行时允许响应终端，所以如果 tasklet 和中断处理程序之间共享了某些数据，必须做好预防工作（比如屏蔽中断，然后获取一个锁）。同样如果 tasklet 和其他 tasklet 或 软中断 之间共享了数据，必须进行适当的锁保护。

调度自己的 tasklet
通过调用 taskelet_schedule() 函数并传递给他相应的 tasklet_struct 的指针，该函数就会被调度以便执行：
tasklet_schedule(my_tasklet);
如果相同的 tasklet 被调度了两次，那他只会运行一次，但是如果这时它已经开始运行了，比如在另一个处理器上，那么这个新的tasklet会被重新调度并再次运行。作为一种优化措施，一个 tasklet 总在调度它的处理器上执行--这是希望能更好的利用处理器的告诉缓存。
可以调用 tasklet_disable() 函数来禁止某个指定的 tasklet。如果该 tasklet 当前正在执行，这个函数会等到它执行完毕在返回。也可以调用 tasklet_disable_nosync() 函数，他也用来禁止指定的 tasklet，不过它无需在返回前等待 tasklet 执行完毕。这么做往往不太安全。同样可以调用 tasklet_enable() 函数激活一个 tasklet， 如果希望激活 DECLARE_TASKLET_DISABLED() 创建的 tasklet，也可以调用这个函数，例如：
tasklet_disable(&my_tasklet);
tasklet_enable(my_tasklet);
可以通过调用 tasklet_kill() 函数从挂起的队列中去掉一个 tasklet 。在处理一个经常重新调度他自身的 tasklet 的时候，这个操作会很有用。这个函数首先等待该 tasklet 执行完毕，然后再将它移除。所以不能在中断上下文中使用。

##### ksoftirqd
在内核中实现的方案是触发后不会立即重新执行软中断，作为改进，当大量软中断出现的时候，内核会唤醒一组内核线程来处理这些负载。这些线程在最低的优先级上运行（nice值为19），这能避免他们跟其他重要的任务抢夺资源，但是他们最终也一定会被执行。每个处理器都有一个这样的线程。所有线程的名字都叫做 ksoftirq/n ，n是对应处理器的编号。只要有待处理的软中断，ksoftirq 就会调用 do_softirq() 去处理他们呢。通过重复执行这样的操作，重新触发的软中断也会执行。当所有需要执行的操作完成后，该内核线程将自己设置为 TASK_INTERRUPTBLE 状态，唤起调度程序选择其他可执行进程投入运行。只要 do_softirq() 函数发现已经执行过的内核线程重新触发了自己，软中断内核线程就会被唤醒。



#### 工作队列
##### 机制
前面介绍的可延迟函数运行在中断上下文中，于是导致了一些问题，说明它们不可挂起，也就是说软中断不能睡眠、不能阻塞，原因是由于中断上下文处于内核态，没有进程切换，所以如果软中断一旦睡眠或者阻塞，将无法退出这种状态，导致内核会整个僵死。因此，可阻塞函数不能用软中断来实现。但是它们往往又具有可延迟的特性。而且由于是串行执行，因此只要有一个处理时间较长，则会导致其他中断响应的延迟。为了完成这些不可能完成的任务，于是出现了工作队列，它工作在进程上下文，能够在不同的进程间切换，以完成不同的工作。

工作队列能运行在进程上下文，它将工作给一个内核线程，作为中断守护线程来使用。多个中断可以放在一个线程中，也可以每个中断分配一个线程。我们用结构体workqueue\_struct表示工作者线程，工作者线程是用内核线程实现的。而工作者线程是如何执行被推后的工作——有这样一个链表，它由结构体work\_struct组成，而这个work\_struct则描述了一个工作，一旦这个工作被执行完，相应的work\_struct对象就从链表上移去，当链表上不再有对象时，工作者线程就会继续休眠。因为工作队列是线程，所以我们可以使用所有可以在线程中使用的方法。

系统允许有许多种类型的工作者线程存在。对于指定的一个类型，系统的每个CPU上都有一个该类的工作者线程。内核中有些部分可以根据需要来创建工作者线程，而在默认情况下内核只有event这一种类型的工作者线程。而 workqueue_struct 结构体则表示给定类型的所有工作者线程。

```c
struct workqueue_struct {   
    struct list_head    pwqs;/* WR: all pwqs of this wq */     
    struct list_head    list;/* PR: list of all workqueues */     
    
    struct mutex        mutex;/* protects this wq */     
    int work_color; /* WQ: current work color */     
    int flush_color;/* WQ: current flush color */     
    atomic_t        nr_pwqs_to_flush; /* flush in progress */     
    struct wq_flusher   *first_flusher; /* WQ: first flusher */     
    struct list_head    flusher_queue;/* WQ: flush waiters */     
    struct list_head    flusher_overflow; /* WQ: flush overflow list */     
    
    struct list_head    maydays;/* MD: pwqs requesting rescue */     
    struct worker       *rescuer;/* MD: rescue worker */     
    
    int nr_drainers;/* WQ: drain in progress */     
    int saved_max_active; /* WQ: saved pwq max_active */     
    
    struct workqueue_attrs  *unbound_attrs; /* PW: only for unbound wqs */     
    struct pool_workqueue   *dfl_pwq;/* PW: only for unbound wqs */     
    
    #ifdef CONFIG_SYSFS     
    struct wq_device    *wq_dev;/* I: for sysfs interface */     
    #endif     
    #ifdef CONFIG_LOCKDEP     
    char *lock_name;   
    struct lock_class_key   key;   
    struct lockdep_map  lockdep_map;   
    #endif     
    char name[WQ_NAME_LEN]; /* I: workqueue name */     
    
    /*     
    \* Destruction of workqueue_struct is RCU protected to allow walking     
    \* the workqueues list without grabbing wq\_pool\_mutex.     
    \* This is used to dump all workqueues from sysrq.     
    */     
    struct rcu_head     rcu;   
    
    /* hot fields used during command issue, aligned to cacheline */     
    unsigned   int flags ____cacheline_aligned; /* WQ: WQ_* flags */     
    struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */     
    struct pool_workqueue __rcu *numa_pwq_tbl[]; /* PWR: unbound pwqs indexed by node */     
};

struct work_struct {   
    atomic_long_t data; 
    struct list_head entry;   
    work_func_t func; 
    #ifdef CONFIG_LOCKDEP     
    struct lockdep\_map lockdep\_map;   
    #endif
};
```


##### 使用
```c
静态创建工作队列：
DECLARE_WORK(name, void(*func)(void *), void *data)

定义一个工作队列：
struct work_struct my_wq;

定义一个处理函数：
void my_wq_func(struct work_struct *work);
函数执行在进程上下文中，默认情况下允许响应中断，并且不支持有任何锁，如果需要，函数可以睡眠。需要注意的是，尽管操作处理函数运行在进程上下文，但它不能访问用户空间，因此内核线程在用户空间没有相关的内存映射。在发生系统调用时，内核会代表用户空间的进程运行，此时它才能访问用户空间，也只有在此时他才会映射用户空间的内存

初始化工作队列并将工作队列与处理函数绑定：
INIT_WORK(&my_wq, my_wq_func);

调度工作队列，把指定的工作的处理函数提交给缺省的 events 工作线程：
schedule_work(&my_wq);

延迟一段时间后再执行：
schedule_delayed_work(&work, delay);

flush 操作：
void flush_scheduled_work(void);
函数会一直等待，直到队列中所有对象都被执行后才返回。在等待所有待处理的工作执行的时候，该函数会进入休眠状态，所以只能在进程上下文中使用它。
注意：该函数并不取消任何延迟执行的工作。就是说，任何通过 schedule_delayed_work() 调度的工作，如果其延迟时间未结束，他并不会因为调用 schedule_delayed_work() 而被取消掉

取消延迟执行的工作：
int cancel_delayed_work(struct work_struct *work);

创建新的工作队列：
如果缺省的队列不能满足需要，应该创建一个新的工作队列和与之对应的工作者线程。由于这么做会在每个处理器上都创建一个工作者线程，所以只有在你明确了必须要靠自己的一套线程来提高性能的情况下，再创建自己的工作队列。
创建一个新的任务队列和与之相关的工作者线程：
struct workqueue_struct *keventd_wq;
keventd_wq = create_workqueue("events");  // events 工作线程，如果自己创建线程要改成别的名字

调度，与 schedule_work 以及 schedule_delayed_work 相近：
static inline bool queue_work(struct workqueue_struct *wq, struct work_struct *work)
static inline bool queue_delayed_work(struct workqueue_struct *wq, struct delayed_work *dwork, unsigned long delay)

flush 操作，与 flush_scheduled_work() 作用相同：
void   flush_workqueue(struct workqueue_struct *wq)
```


#### thread_irq
内核中除了可以通过request_irq()、devm_request_irq()申请中断以外，还可以通过 request_threaded_irq() 和 devm_request_threaded_irq() 申请：
```c
int   request_threaded_irq(unsigned   int irq, 
                           irq_handler_t handler,     // 对应的函数执行在中断上下文，如果 handler 结束时返回值是 IRQ_WAKE_THREAD，内核会调度对应线程执行 thread_fn 对应的函数
                                                      // 该参数可以传入 NULL，这种情况下内核会用默认的 irq_default_primary_handler() 代替 handler，并会使用 IRQF_ONESHOT 标记
                           irq_handler_t thread_fn,   // 对应的函数执行在内核线程
                           unsigned   long irqflags,  // 可以设置 IRQF_ONESHOT 标记，这样内核会自动帮助我们在中断上下文屏蔽对应的中断号，而在内核调度 thread_fn 执行后，重新使能该中断号
                           const   char *devname,
                           void *dev_id)

devm_request_threaded_irq 的参数含义与 request_threaded_irq 一致
int   devm_request_threaded_irq(struct device *dev,
                                unsigned   int irq,   
                                irq_handler_t handler, 
                                irq_handler_t thread_fn, 
                                unsigned   long irqflags,
                                const   char *devname,  
                                void *dev_id)
```
用这两个api申请中断的时候，内核会为相应的中断号分配一个对应的内核线程。注意这个线程只针对这个中断号，如果其他中断也通过 request_threaded_irq 申请，自然也会得到新的内核线程。


#### 如何选择下半部机制

1.  软中断和tasklet运行在中断上下文，工作队列运行在进程上下文。如果需要休眠则选择工作队列，否则选择tasklet；如果对性能要求较高则选择软中断。
2.  从易用性考虑，首选工作队列，然后tasklet，最后是软中断，因为软中断需要静态创建。
3.  从代码安全考虑，如果对底半部代码保护不够安全，则选择tasklet，因为相对软中断，tasklet对锁要求低，上面也简述它们工作方式以及运用场景。
