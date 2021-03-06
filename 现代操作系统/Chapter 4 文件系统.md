## 文件系统

长期存储信息的三个基本要求：

1. 能够存储大量信息
2. 使用信息的进程终止时，信息仍旧存在
3. 必须能使多个进程并发访问有关信息

**文件**是进程创建的信息逻辑单元。

进程可以读取已经存在的文件，并在需要时建立新的文件。存储在文件中的信息必须是持久的。

操作系统中处理文件的部分称为文件系统。

### 文件

#### 文件命名

在进程创建文件时命名，在进程终止时，该文件仍旧存在，并且其他进程可以通过这个文件名对它进行访问。

文件名圆点后面的部分称为文件扩展名，通常表示文件的一些信息。

#### 文件结构

- 字节结构
- 记录结构
- 树

#### 文件类型

- 普通文件：包含有用户信息的文件
- 目录：管理文件系统结构的系统文件
- 字符特殊文件：用于串行 I/O 设备
- 块特殊文件：用于磁盘类设备

普通文件一般分为 ASCII 文件和二进制文件。

ASCII 文件的最大优势是可以显示和打印，还可以用任何文本编辑器进行编辑。另外如果很多程序都以 ASCII 文件作为输入和输出，就很容易把一个程序的输出作为另一个程序的输入。

二进制文件打印出来是无法理解的、充满混乱的一张表。通常二进制文件有一定的内部结构，使用该文件的程序才了解这种结构。只有文件格式正确时，操作系统才会执行这个文件。

早期的 UNIX 二进制文件有五个段：文件头、正文、数据、重定位位及符号表。文件头以魔数开始，表明该文件是一个可执行的文件。魔数后面是文件中各段的长度、执行的起始位置和一些标志位。程序本身的正义和数据在文件头后面。这些被装入内存，并使用重定位位重新定位。

#### 文件访问

- 顺序访问
- 随机访问文件

#### 文件属性

如文件创建的日期和时间、文件大小等，这些附加信息称为文件属性，也称为元数据。

#### 文件操作

- create
- delete
- open
- close
- read
- write
- append：只能在文件末尾添加数据
- seek：对于随机访问文件，要指定从何处开始获取数据，通常的方法是用 seek 系统调用把当前位置指针指向文件的特殊位置
- get attributes
- set attributes
- rename

### 目录

#### 一级目录系统

在一个目录中包含所有的文件，有时也称为根目录。优点是简单并且能够快速定位文件。

#### 层次目录系统

通过这种方式可以用很多目录把文件以自然的方式分组。

#### 路径名

- 绝对路径名：从根目录到文件的路径组成。
- 相对路径名：常和工作目录（当前目录）一起使用。

#### 目录操作

- create
- delete
- closedir
- readdir
- rename
- link：链接技术允许在多个目录中出现同一个文件
- unlink：删除目录项，即删除指定目录下的文件

### 文件系统的实现

#### 文件系统布局

多数磁盘划分为一个或多个分区，每个分区中有一个独立的文件系统。磁盘的 0 号扇区称为主引导记录（Master Boot Record），用来引导计算机。在 MBR 的结尾是分区表。该表给出了每个分区的起始和结束地址。

每个分区都从一个引导块开始，即使它不含有一个可启动的操作系统。接着包含：

- 超级块：包含文件系统的所有关键参数，如确定文件系统类型的魔数、文件系统中块的数量以及其他重要的管理信息。
- 空闲块
- i 节点：是一个数据结构数组，每个文件一个，用来说明文件的方方面面
- 根目录
- 文件和目录

#### 文件的实现

关键问题是记录各个文件分别用到哪些磁盘块。

1. 连续分配

    最简单的分配方案是把每个文件作为一连串连续数据块存储在磁盘上。

    实现简单；读操作性能较好，不再需要寻道和旋转延迟。但是随着时间推移，磁盘空间会变得零碎。

2. 链表分配

    为每个文件构造磁盘块链表，每个块的第一个字作为指向下一块的指针，块的其他部分存放数据。

    可以充分利用每个磁盘块。但随机访问相当缓慢；指针占去了一些字节，每个磁盘块存储数据的字节数不再是 2 的整数次幂，怪异的数据大小降低了系统的运行效率，因为许多程序都是以长度为 2 的整数次幂来读写磁盘块的。

3. 采用内存中的表进行链表分配

    取出每个磁盘块的指针字，把它们放在内存的一个表中，这个表称为文件分配表（File Allocation Table）。

    主要缺点是必须把整个表都存放在内存中，不能较好地扩展并应用于大型磁盘中。

4. i 节点

    给每个文件赋予一个称为 i 节点的数据结构，其中列出了文件属性和文件块的磁盘地址，用来记录各个文件分别包含哪些磁盘块。

    只有在对应文件打开时，其 i 节点才在内存中。

#### 目录的实现

目录系统的主要功能是把 ASCII 文件名映射成定位文件数据所需的信息。目录项中提供了查找文件磁盘块所需要的信息，因系统而异，这些信息有可能是整个文件的磁盘地址、第一个块的编号或者是 i 节点。

应该在何处存放文件属性：

- 把文件属性直接存放在目录项中
- 对于采用 i 节点的系统，把文件属性存放在 i 节点中，这样目录项会更短

对于可变长度的文件名：

- 每个目录项有一个固定部分，后面是一个任意长度的实际文件名。但是当移走文件后，就引入了一个长度可变的空隙，而下一个进来的文件不一定正好适合这个空隙。而且一个目录项可能会分布在多个页面上，在读取文件名时可能发生缺页中断。
- 使目录项自身都有固定长度，而将文件名放置在目录后面的堆中。

为了加快查找文件名速度的方法是在每个目录中使用散列表或者将查找结果放入高速缓存中。

#### 共享文件

如果一个共享文件同时出现在属于不同用户的不同目录下，工作起来就很方便。这样，文件系统本身是一个有向无环图而不是树。

#### 日志结构文件系统

磁盘的寻道时间没有快速发展导致了性能瓶颈。

将整个磁盘结构化为一个日志，每隔一段时间，或是有特殊需要时，被缓冲在内存中的所有未决的写操作都被放到一个单独的段中，作为在日志末尾的一个邻接段写入磁盘。

#### 日志文件系统

保存一个用于记录系统下一步将要做什么的日志，这样系统在完成它们即将完成的任务前崩溃时，重新启动后可以查看日志，获取崩溃前计划完成的任务并完成它们。被写入日志的操作必须是幂等的。为了增加可靠性，文件系统可以引入原子事务的概念。

#### 虚拟文件系统

关键思想是抽象出所有文件系统都共有的部分，并且将这部分代码放在单独的一层，该层调用底层的实际文件系统来具体管理数据。

### 文件系统管理和优化

#### 磁盘空间管理

1. 块大小

    拥有大的块尺寸会导致小的文件浪费了大量的磁盘空间。另一方面，小的块尺寸意味着大多数文件会跨越多个块，因此需要多次寻道与旋转延迟才能读出它们，从而降低了性能。

2. 记录空闲块

    - 采用磁盘块链表，链表的每个块中包含尽可能多的空闲磁盘块号。
    - 采用位图，在位图中，空闲块用 1 表示，已分配块用 0 表示。

    如果空闲块倾向于成为一个长的连续分块的话，则空闲列表系统可以改成记录连续分块而不是单个的块。在最好的情况下，一个基本上空的磁盘可以用空闲块的地址和空闲块的计数来表达。但是如果磁盘产生了很严重的碎片，记录连续分块会比记录单独的块效率要低。

3. 磁盘配额

    系统管理员分给每个用户拥有文件和块的最大数量，操作系统确保每个用户不超过分给他们的配额。

#### 文件系统备份

- 从意外的灾难中恢复
- 从错误的操作中恢复

1. 为文件做备份既耗时间又费空间，合理的做法是只备份特定目录及其下的全部文件，而不是整个文件系统。
2. 对前一次备份以来没有更改过的文件再做备份是一种浪费，因而产生了增量转储的思想。最简单的增量转储形式就是周期性地做全面的转储，而每天只对从上一次全面转储起发生变化的数据做备份。这种做法极大地缩减了转储时间，但恢复起来却更复杂，因为最近的全面转储先要全部恢复，随后按逆序进行增量转储。
3. 转储的往往是海量数据，那么在将其写入之前对文件进行压缩就很有必要。
4. 对活动文件系统做备份是很难的，可能会导致文件系统的不一致性。
5. 做备份会给一个单位引入许多非技术性问题。

转储的方案有：

- 物理转储

    从磁盘的第 0 块开始，将全部的磁盘块按序输出到磁带上，直到最后一块复制完毕。

    未使用的磁盘块无须备份；防止在对坏块文件备份时的无止境磁盘读错误。

    优点是简单、极为快速；主要缺点是，既不能跳过选定的目录，也无法增量存储，还不能满足恢复个人文件的请求。因此，绝大多数配置都使用逻辑转储。

- 逻辑转储

    从一个或几个指定的目录开始，递归地转储其自给定基准日期后有所更改的全部文件和目录。所以在逻辑转储中，转储磁带上会有一连串精心标识的目录和文件，这样就很容易满足恢复特定文件或目录的请求。

#### 文件系统的一致性

计算机带有一个实用程序以检验文件系统的一致性。

一致性检查分为两种：

- 块的一致性检查
- 文件的一致性检查

#### 文件系统性能

1. 高速缓存
2. 块提前读
3. 减少磁盘臂运动

#### 磁盘碎片整理

移动文件使它们相邻，并把所有的空闲空间放在一个或多个连续的区域内。