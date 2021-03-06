---
计算机知识: 用户态和内核态
---

# 用户态/内核态

## 简介

linux 驱动程序一般工作在内核空间， 但是也可以工作在用户空间。 下面我们将详细解析， 什么是内核空间，什么是用户空间， 以及如何判断他们。

Linux 简化了分段机制， 似的虚拟地址与线性地址总是一致， 因此Linux的虚拟地址空间也分为0~4G \(32位情况下\),Linux 内核将这4G字节的空间分为2部分。 将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF），供内核使用，称为“内核空间”。而将较低的3G字节（从虚拟地址 0x00000000到0xBFFFFFFF），供各个进程使用，称为“用户空间）。

因为每个进程可以通过系统调用进入内核，因此，Linux内核由系统内的所有进程共享。于是，从具体进程的角度来看，每个进程可以拥有4G字节的虚拟空间。

linux 使用两级保护机制， 0级供内核使用， 3级供用户程序使用。 每个进程有各自的私有用户空间（0-3G）这个空间对系统中的其他进程是不可见的。 最高的1GB字节虚拟内核空间则为所有进程以及内核所**共享**

内核空间中存放的是**内核代码和数据**，而进程的用户空间中存放的是**用户程序的代码和数据** , 不管是内核空间还是用户空间， 他们都处于虚拟空间中， 虽然内核空间占据每个虚拟空间中的最高1GB字节， 但是映射到物理内存却总是从最低地址（0x00000000）开始， 对内核空间来说， 其地址映射是很简单的线性映射。

内核空间和用户空间之间如何进行通信？

内核空间和用户空间一般通过系统调用进行通信。

如何判断一个驱动是用户模式驱动还是内核模式驱动， 判断的标准是什么？

用户空间模式的驱动一般通过系统调用完成对硬件的访问， 如通过系统调用将驱动的io空间映射到用户空间等。因此主要的判断依据就是系统调用，

内核空间和用户空间上有太多的不同， 比如用户态的链表和内核链表不一样。

用户态每个应用程序空间是虚拟的， 相对独立的， 内核态中却不是是独立的， 所以变成要非常小心等等。

还有用户态和内核态程序通讯的方法很多， 不单单是系统调用， 实际上系统调用是一个不好的选择， 因为需要系统调用号， 这个需要统一分配。

可以通过ioctl ， sysfs，proc 等来完成。

## 内核态和用户态

当一个进程执行系统调用而陷入内核代码中执行时候， 我们就称进程处于内核运行态 或者简称 内核态。 此时处理器出去特权级最高的（0级）内核代码中执行。 当进程处于内核态时候， 执行的内核代码会使用当前进程的内核栈，每个进程都有自己的内核栈。当进程在执行用户自己的代码时候 ，则称其处于用户运行态（用户态）， 即此时处理器在特权等级最低的（3级） 用户代码中运行， 当正在执行用户程序而突然被中断程序中断时候， 此时用户程序也可以象征性的成为处于进程的内核态。 因为中断处理程序将使用当前进程的内核栈。 这与处于内核态进程的状态有些类似。

### 内核栈

在C语言书里面讲的栈， 堆， 大部分都是用户态的概念。 用户态的堆栈，对应用户进程虚拟地址空间的一个区域， 栈向下增长， 堆用malloc 分配， 向上增长。

用户空间的堆栈， 在task\_struct -&gt; mm -&gt; vm\_area 里面描述，都是属于进程虚拟地址空间的一个区域

而内核态的栈在task\_struct -&gt; stack 里面描述， 其底部是thread\_info 对象， thread\_info 可以用来快速获取task\_struct对象， 整个stack区域一般只有一个内存页面， 32位机器也就4KB

所以说 一个进程的内核栈 也是进程私有的， 只是在task\_struct -&gt; stack Lim 获取。

内核态里没有进程堆的概念。

## 进程上下文和中断上下文

处理器总是处于以下状态中的一种：

* 内核态， 运行于进程上下文， 内核代表进程运行于内核空间
* 内核态， 运行于中断上下文， 内核代表英健运行于内核空间
* 用户态， 运行于用户空间

用户空间的应用程序， 通过系统调用， 进入内核空间。 这时候用户空间的进程要传递很多变量， 参数的值给内核。 内核态运行的时候也要保存用户进程的一些寄存器，变量等等。 所谓的进程上下文，可以看做是用户进程传递给内核的这些参数以及内核要保存的那一整套的变量和寄存器只和当时的环境等等。

硬件通过出发信号， 导致内核调用中断处理程序， 进入内核空间， 这个过程中， 硬件的一些变量和参数也要传递给内核。 内核通过这些参数进行中断处理， 所谓的中断上下文， 其实也看做是硬件传递过来的这些参数和内核需要保存的一些其他环境。

## 内核虚拟空间布局

在多任务操作系统中，每个进程都运行在属于自己的内存沙盘中， 这个沙盘就是虚拟地址空间（virtual address space）在32位模式下， 它是一个4GB的内存地址， 在Linux 系统中 内核进程和用户进程所占的虚拟内存比是1：3 而window 系统为2：2 ， 这并不意味着内核使用那么多的物理内存， 仅仅表示它可以支配这部分地址空间， 根据需要映射到物理内存。

虚拟地址通过页表\(Page Table\)映射到物理内存，页表由操作系统维护并被处理器引用。内核空间在页表中拥有较高特权级，因此用户态程序试图访问这些页时会导致一个页错误\(page fault\)。在Linux中，内核空间是持续存在的，并且在所有进程中都映射到同样的物理内存。内核代码和数据总是可寻址，随时准备处理中断和系统调用。与此相反，用户模式地址空间的映射随进程切换的发生而不断变化。

Linux进程在虚拟内存中的标准内存段布局如下图所示：

![](../../.gitbook/assets/image%20%288%29.png)

其中用户地址空间的蓝色条带对应于映射到物理内存的不同内存段， 灰白区域表示为映射的部分， 这些段只是简单的内存地址范围， 与Intel处理器的段没有关系。

上图中Random stack offset 和Random mmap offset 等随机值 移栽防止恶意程序， Linux 通过对栈， 内存映射段， 堆的其实地址加上随机偏移量来打乱布局， 以免恶意程序通过计算访问栈， 库函数等地址。

用户进程部分 分段存储内容如下表所示（按地址递减顺序）

| 名称 | 存储内容 |
| :--- | :---: |
| 栈 | 局部变量， 函数参数， 返回地址等 |
| 堆 | 动态分配的内存 |
| BSS段 | 未初始化或者初值为0的全局变量或者静态局部变量 |
| 数据段 | 已经初始化并且初值非零的全局变量和静态局部变量 |
| 代码段 | 可执行代码，字符串字面值， 只读变量 |

在讲应用程序加载到内存空间执行时候， 操作系统负责代码段， 数据段， 和BSS段的加载， 并在内存中为这些段分配空间。 栈也有操作系统分配和管理。 堆由程序员自己管理， 即显示的申请和释放空间。

## 内核空间

内核总是驻留在内存中，是操作系统的一部分， 内核空间为内核保留， 不允许应用程序读写该区域的内容或者直接调用内核代码定义的函数。

## 栈

栈又称为堆栈， 由编译器自动分配释放， 行为类似数据结构中的栈， 堆栈主要有三个用途

* 为函数内部声明的非静态局部变量提供存储空间
* 记录函数调用过程相关的维护信息， 成为栈帧（stack frame）或者过程活动记录（procedure activation record）它包括函数返回地址， 不适合装入寄存器的函数以及一些寄存器的保存。 除递归调用以外，堆栈并非必要。
* 临时存储区， 用于暂存长算术表达式部分计算结果 或者alloca（）函数分配的栈内存。

## 内存映射段 mmap

此外内核将硬盘文件的内容直接映射到内存， 任何应用程序都可以通过linux 的mmap 系统调用请求这种映射， 内存映射是一种方便高效的文件IO形式， 因为被用于装载动态共享库。用户也可以创建匿名内存映射， 该映射没有对应的文件，可用于存放程序数据。 在linux 中， 若公共malloc 请求一大块内存，C运行库将创建一个匿名的内存映射， 而不是用堆内存，”大块” 意味着比阈值 MMAP\_THRESHOLD还大，缺省为128KB，可通过mallopt\(\)调整。

该区域 用于映射可执行文件用到的动态链接库，

## 堆

在kernel 2.6 的32位Linux 系统中， malloc申请的最大内存理论值在2.9G左右

堆用于存放进程运行时候动态分配的内存段， 可动态扩展或者缩减， 堆中的内容是匿名的， 不能按照名字直接访问， 只能通过指针间接访问， 当进程调用malloc（c）等函数分配内存时候， 新分配的内存动态添加到堆上。 当调用free（c）等函数释放内存时候， 被释放的内存从堆中剔除。

分配的堆内存是经过字节对齐的空间， 以适合原子操作，堆管理器通过链表管理每个申请的内存。由于堆申请和释放是无序的， 最终会产生内存碎片。堆内存一般由应用程序分配和释放。 回收的内存可供重新使用。 若程序员不释放， 程序结束时候， 操作系统可能会自动释放。

堆的末端又break 指针标示， 当堆管理器需要更多内存时候， 可通过系统调用brk\(\) 和sbrk\(\) 来移动指针以扩张堆，一般由系统自动调用。

## BSS 段

BSS 段中通常存放程序中以下符号。

* 未初始化的全局变量和静态局部变量
* 初始值为0的全局变量和静态局部变量
* 为定义切初值不为0 的符号（该初值即common block 的大小）

C语言中，未显式初始化的静态分配变量被初始化为0\(算术类型\)或空指针\(指针类型\)。由于程序加载时，BSS会被操作系统清零，所以未赋初值或初值为0的全局变量都在BSS中。BSS段仅为未初始化的静态分配变量预留位置，在目标文件中并不占据空间，这样可减少目标文件体积。但程序运行时需为变量分配内存空间，故目标文件必须记录所有未初始化的静态分配变量大小总和\(通过start\_bss和end\_bss地址写入机器代码\)。当加载器\(loader\)加载程序时，将为BSS段分配的内存初始化为0。在嵌入式软件中，进入main\(\)函数之前BSS段被C运行时系统映射到初始化为全零的内存\(效率较高\)。

```text
 注意，尽管均放置于BSS段，但初值为0的全局变量是强符号，而未初始化的全局变量是弱符号。若其他地方已定义同名的强符号(初值可能非0)，则弱符号与之链接时不会引起重定义错误，但运行时的初值可能并非期望值(会被强符号覆盖)。因此，定义全局变量时，若只有本文件使用，则尽量使用static关键字修饰；否则需要为全局变量定义赋初值(哪怕0值)，保证该变量为强符号，以便链接时发现变量名冲突，而不是被未知值覆盖。

 某些编译器将未初始化的全局变量保存在common段，链接时再将其放入BSS段。在编译阶段可通过-fno-common选项来禁止将未初始化的全局变量放入common段。

 此外，由于目标文件不含BSS段，故程序烧入存储器(Flash)后BSS段地址空间内容未知。U-Boot启动过程中，将U-Boot的Stage2代码(通常位于lib_xxxx/board.c文件)搬迁(拷贝)到SDRAM空间后必须人为添加清零BSS段的代码，而不可依赖于Stage2代码中变量定义时赋0值。
```

## 数据段\(Data\)

```text
 数据段通常用于存放程序中已初始化且初值不为0的全局变量和静态局部变量。数据段属于静态内存分配(静态存储区)，可读可写。

 数据段保存在目标文件中(在嵌入式系统里一般固化在镜像文件中)，其内容由程序初始化。例如，对于全局变量int gVar = 10，必须在目标文件数据段中保存10这个数据，然后在程序加载时复制到相应的内存。

 数据段与BSS段的区别如下： 

 1) BSS段不占用物理文件尺寸，但占用内存空间；数据段占用物理文件，也占用内存空间。

 对于大型数组如int ar0[10000] = {1, 2, 3, ...}和int ar1[10000]，ar1放在BSS段，只记录共有10000*4个字节需要初始化为0，而不是像ar0那样记录每个数据1、2、3...，此时BSS为目标文件所节省的磁盘空间相当可观。

 2) 当程序读取数据段的数据时，系统会出发缺页故障，从而分配相应的物理内存；当程序读取BSS段的数据时，内核会将其转到一个全零页面，不会发生缺页故障，也不会为其分配相应的物理内存。

 运行时数据段和BSS段的整个区段通常称为数据区。某些资料中“数据段”指代数据段 + BSS段 + 堆。
```

## 代码段\(text\)

```text
 代码段也称正文段或文本段，通常用于存放程序执行代码(即CPU执行的机器指令)。一般C语言执行语句都编译成机器代码保存在代码段。通常代码段是可共享的，因此频繁执行的程序只需要在内存中拥有一份拷贝即可。代码段通常属于只读，以防止其他程序意外地修改其指令(对该段的写操作将导致段错误)。某些架构也允许代码段为可写，即允许修改程序。

 代码段指令根据程序设计流程依次执行，对于顺序指令，只会执行一次(每个进程)；若有反复，则需使用跳转指令；若进行递归，则需要借助栈来实现。

 代码段指令中包括操作码和操作对象(或对象地址引用)。若操作对象是立即数(具体数值)，将直接包含在代码中；若是局部数据，将在栈区分配空间，然后引用该数据地址；若位于BSS段和数据段，同样引用该数据地址。

 代码段最容易受优化措施影响。
```

## 保留区

```text
 位于虚拟地址空间的最低部分，未赋予物理地址。任何对它的引用都是非法的，用于捕捉使用空指针和小整型值指针引用内存的异常情况。

 它并不是一个单一的内存区域，而是对地址空间中受到操作系统保护而禁止用户进程访问的地址区域的总称。大多数操作系统中，极小的地址通常都是不允许访问的，如NULL。C语言将无效指针赋值为0也是出于这种考虑，因为0地址上正常情况下不会存放有效的可访问数据。

 在32位X86架构的Linux系统中，用户进程可执行程序一般从虚拟地址空间0x08048000开始加载。该加载地址由ELF文件头决定，可通过自定义链接器脚本覆盖链接器默认配置，进而修改加载地址。0x08048000以下的地址空间通常由C动态链接库、动态加载器ld.so和内核VDSO(内核提供的虚拟共享库)等占用。通过使用mmap系统调用，可访问0x08048000以下的地址空间。
```

