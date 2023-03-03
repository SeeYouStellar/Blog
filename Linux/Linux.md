### 进程管理命令

https://sites.google.com/site/linuxxuexi/rhel-xi-tong-guan-li/di7zhanglinux-xi-tong-jin-cheng-guan-li

### Linux进程空间分布概述

参考博客https://blog.csdn.net/jinking01/article/details/106825769

1. BSS段：BSS段（bss segment）通常是指用来存放程序中**未初始化的全局变量**的一块内存区域。BSS是英文Block Started by Symbol的简称。BSS段属于静态内存分配。
2. 数据段：数据段（data segment）通常是指用来存放程序中**已初始化的全局变量**的一块内存区域。数据段属于静态内存分配。
3. 代码段：代码段（code segment/text segment）通常是指用来存放**程序执行代码**的一块内存区域。这部分区域的大小在程序运行前就已经确定，并且内存区域通常属于只读, 某些架构也允许代码段为可写，即允许修改程序。在代码段中，也有可能包含一些**只读的常数变量**，例如字符串常量等。
4. 堆（heap）：堆是用于存放进程运行中被动态分配的内存段，它的大小并不固定，可动态扩张或缩减。当进程调用**malloc**等函数分配内存时，新分配的内存就被动态添加到堆上（堆被扩张）；当利用**free**等函数释放内存时，被释放的内存从堆中被剔除（堆被缩减）
5. 栈(stack)：栈又称堆栈， 是用户存放程序临时创建的局部变量，也就是说我们函数括弧“{}”中定义的变量（但不包括static声明的变量，static意味着在数据段中存放变量）。除此以外，在函数被调用时，其参数也会被压入发起调用的进程栈中，并且待到调用结束后，函数的返回值也会被存放回栈中。由于栈的先进后出特点，所以栈特别方便用来保存/恢复调用现场。从这个意义上讲，我们可以把堆栈看成一个寄存、交换临时数据的内存区。
6. 内存映射段：在栈的下方是内存映射段，内核将文件的内容直接映射到内存。任何应用程序都可以通过Linux的mmap()系统调用或者Windows的CreateFileMapping()/MapViewOfFile()请求这种映射。内存映射是一种方便高效的文件I/O方式，所以它被用来加载动态库。创建一个不对应于任何文件的匿名内存映射也是可能的，此方法用于存放程序的数据。在Linux中，如果你通过malloc()请求一大块内存，C运行库将会创建这样一个匿名映射而不是使用堆内存。“大块”意味着比MMAP_THRESHOLD还大，缺省128KB，可以通过mallocp()调整。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWM0LnpoaW1nLmNvbS84MC92Mi04MmY1NjcxNDZmODM2NDViZGZhZGVhMDY4MWFkMDgxZl83MjB3LnBuZw?x-oss-process=image/format,png)

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWMzLnpoaW1nLmNvbS84MC92Mi1lOTBmYjA0OWVlNmY3OWRmYThkZGZlOTAzNDkwM2Y5Ml83MjB3LnBuZw?x-oss-process=image/format,png)

注意图中地址方向，地址从小到大，依次是代码段，数据段，BSS段，堆，栈（堆顶小地址，类似大端存储）

**堆和栈区别**

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWMxLnpoaW1nLmNvbS84MC92Mi1lMDdkMzhhNDAzMTc1Y2I3NjJhZTliMGI0MmFlNDJlNF83MjB3LnBuZw?x-oss-process=image/format,png)

### 内核空间和用户空间

内核空间中存放的是内核代码和数据，而进程的用户空间中存放的是用户程序的代码和数据。

通常32位Linux内核地址空间划分0~3G为用户空间，3~4G为内核空间

### 挂载mount

因为Linux系统将所有的硬件设备都当做文件来处理，当使用光驱等硬件设备时，就必须将其挂载到系统中，只有这样Linux才能识别。也就是所谓的Linux系统“一切皆文件”，所有文件都放置在以根目录为树根的树形目录结构中。在 Linux 看来，任何硬件设备也都是文件，它们各有自己的一套文件系统（文件目录结构）。
当在 Linux 系统中使用这些硬件设备时，只有将Linux本身的文件目录与硬件设备的文件目录合二为一，硬件设备才能为我们所用。合二为一的过程称为“挂载”。