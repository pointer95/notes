## 输入/输出

### I/O 硬件原理

#### I/O 设备

- 块设备

    把信息存储在固定大小的块中，每个块都有自己的地址。通常块的大小在 512 字节至 65536 字节之间。

- 字符设备

    以字符为单位发送或接收一个字符流，而不考虑任何块结构。字符设备是不可寻址的，也没有任何寻道操作。

#### 设备控制器

I/O 设备一般由机械部件和电子部件组成。电子部件称为设备控制器或适配器。在个人计算机上，电子部件经常以主板上的芯片或插入扩展槽中的印刷电路板的形式出现。机械部件则是设备本身。

控制器卡上通常有一个连接器，通向设备本身的电缆可以插入到这个连接器中。很多控制器可以操作 2 个、4 个甚至 8 个相同的设备。

控制器与设备之间的接口通常是一个低层次的接口。例如，磁盘可以按每个磁道 2 000 000 个扇区，每个扇区 512 字节进行格式化。然而，实际从驱动器出来的是一个串行的比特流，它以一个**前导符**（preamble）开始，接着是一个扇区中的 4096 位，最后是一个校验和，也称为**错误校正码**。前导符是在对磁盘进行格式化时写上去的，它包括柱面数和扇区号、扇区大小及类似的数据，此外还包含同步信息。

控制器的任务是把串行的位流转换为字节块，并进行必要的错误校正工作。字节块通常首先在控制器内部的一个缓冲区中按位进行组装，然后在对校验和进行校验并证明字节块没有错误后，再将它复制到主内存中。

#### 内存映射 I/O

每个控制器有几个寄存器用来与 CPU 进行通信。通过写入这些寄存器，操作系统可以命令设备发送数据、接收数据、开启或关闭，或者执行某些其他操作。通过读取这些寄存器，操作系统可以了解设备的状态，是否准备好接收一个新的命令等。

除了寄存器以外，许多设备还有一个操作系统可以读写的数据缓冲区。

CPU 与设备的控制寄存器和数据缓冲区进行通信的方式：

- 每个控制寄存器被分配一个 I/O 端口号，这是一个 8 位或 16 位的整数。所有 I/O 端口形成 I/O 空间，并且受到保护使得普通用户程序不能对其访问。使用一条特殊的 I/O 指令，CPU 可以读取控制寄存器 PORT 的内容并将结果存入到 CPU 寄存器中。
- 将所有控制寄存器映射到内存空间中，每个控制寄存器被分配唯一的一个内存地址，并且不会有内存被分配这一地址。这样的系统称为内存映射 I/O。在大多数系统中，分配给控制寄存器的地址位于或者靠近地址空间的顶端。

一种混合的方案是具有内存映射 I/O 的数据缓冲区和具有单独的 I/O 端口控制寄存器。

当 CPU 想要读取一个字的时候，将需要的地址放到总线的地址线上，然后在总线的一条控制线上置起一个 READ 信号，同时还要用到信号线来表明需要的是内存空间还是 I/O 空间。如果是内存空间，内存将响应请求。如果是 I/O 空间，I/O 设备将响应请求。如果只有内存空间，那么每个内存模块和每个 I/O 设备都会将地址线和它所服务的地址范围进行比较，如果地址落在这一范围之内，它就会响应请求。

内存映射 I/O 的优点：

- 对于内存映射 I/O，设备控制器只是内存中的变量，在 C 语言中可以和任何其他变量一样寻址。如果需要特殊的 I/O 指令读写设备控制寄存器，那么访问这些寄存器需要使用汇编代码。
-  不需要特殊的保护机制来阻止用户进程执行 I/O 操作。如果每个设备在地址空间的不同页面上拥有自己的控制寄存器，操作系统只要简单地通过在其页表中包含期望的页面就可以让用户控制特定的设备而不是其他设备。这样可以使不同的设备驱动程序放置在不同的地址空间中，不但可以减小内核的大小，而且可以防止驱动程序之间相互干扰。
-  每一条可以引用内存的指令也可以引用控制寄存器。

内存映射 I/O 的缺点：

- 硬件必须能够针对每个页面有选择性地禁用高速缓存，并且操作系统必须管理选择性缓存，因此增添了额外的复杂性。
- 如果只存在一个地址空间，那么所有的内存模块和所有的 I/O 设备都必须检查所有的内存引用，以便了解由谁作出响应。

#### 直接存储器读取

CPU 需要使用直接存储器（Direct Memory Access，DMA）作为寻址设备控制器来与它们交换数据。

DMA 能够独立于 CPU 访问系统总线，它包含若干个可以被 CPU 读写的寄存器，其中包括一个内存地址寄存器、一个字节计数寄存器和一个或多个控制寄存器。控制寄存器指定要使用的 I/O 端口、传送方向（从 I/O 设备读或写到 I/O 设备）、传送单位以及一次突发传送中要传送的字节数。

DMA 控制器请求传送一个字并且得到这个字，如果 CPU 也想使用总线，它必须等待，这一机制被称为**周期窃取**（cycle stealing）。在块模式中，DMA 控制器通知设备获得总线，发起一连串的传送，然后释放总线，这一操作形式被称为**突发模式**（burst mode）。突发模式比周期窃取效率更高，但如果进行的是长时间突发传送，有可能将 CPU 和其他设备阻塞相当长的周期。

在**飞越模式**（fly-by mode）中，DMA 控制器通知设备控制器直接将数据传送到主存，然后发起第二个总线请求将改字写到它应该去的任何地方。这种方案每传送一个字需要一个额外的总线周期，但是更加灵活，因为它可以执行设备到设备的复制甚至是内存到内存的复制。

#### 中断

当一个 I/O 设备完成它的工作时，它就产生一个中断，通过在分配给它的一条总线信号线上置起信号而产生中断的。该信号被主板上的中断控制芯片检测到，由中断控制芯片决定做什么。

### I/O 软件原理

#### I/O 软件的目标

- 设备独立性（device independence）：应该能够编写出可以访问任意 I/O 设备而无需事先指定设备。

- 统一命名（uniform naming）：一个文件或一个设备的名字应该是一个简单的字符串或一个整数，它不应依赖于设备。
- 错误处理（error handling）：错误应该尽可能地在接近硬件的层面得到处理。当控制器发现了一个读错误时，如果它能够处理那么就应该自己设法纠正这一错误。如果控制器处理不了，那么设备驱动程序应当予以处理，可能只需重新读一次这块数据就正确了。
- 同步（synchronous）和异步（asynchronous）传输：大多数物理 I/O 是异步的，在 CPU 启动传输后便转去做其他工作，直到中断发生。
- 缓冲（buffering）：数据离开一个设备之后通常并不能直接存放到其最终的目的地。缓冲涉及大量的复制工作，并且经常对 I/O 性能有重大影响。

#### 程序控制 I/O

I/O 的最简单形式是让 CPU 做全部工作，这一方法称为程序控制 I/O （programmed I/O）。这种方式十分简单但会出现阻塞。

#### 中断驱动 I/O

#### 使用 DMA 的I/O

本质上，DMA 是程序控制 I/O，只是由 DMA 控制器而不是主 CPU 做全部工作。

### I/O 软件层次

- 中断处理程序

    对于大多数 I/O 而言，中断是令人不愉快的事情并且无法避免的，应当将其深深地隐藏在操作系统内部，以便系统的其他部分尽量不与它发生联系。隐藏它们的最好办法是将启动一个 I/O 操作的驱动程序阻塞起来，直到 I/O 操作完成并且产生一个中断。当中断发生时，中断处理程序将做它必须要做的全部工作以便对中断进行处理。然后，它可以将启动中断的驱动程序解除阻塞。

- 设备驱动程序

    每个连接到计算机上的 I/O 设备都需要某些设备特定的驱动程序来对其进行控制。

- 与设备无关的操作系统软件

    1. 设备驱动程序的统一接口

    2. 缓冲

        双缓冲或循环缓冲

    3. 错误报告

    4. 分配与释放专用设备

        某些设备在任意给定的时刻只能由一个进程使用。这要求操作系统对设备使用的请求进行检查，并根据被请求的设备是否可用来接受或者拒绝这些请求。

    5. 提供与设备无关的块大小

        高层软件只需处理抽象的设备，与物理块大小无关。

- 用户级 I/O 软件
    1. 库过程实现系统调用
    2. 假脱机（spooling）系统指多道程序设计系统中处理独占 I/O 设备的一种方法