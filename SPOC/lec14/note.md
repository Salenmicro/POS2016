第14讲

- - -

### 总体介绍
#### 目标
- 了解第一个用户进程创建过程
- 了解进程管理的实现机制
- 了解系统调用框架的实现机制

#### 练习
- 加载应用程序并执行
- 父进程复制自己的内存空间给子进程
- 分析系统调用和进程管理的实现

#### 流程概述
- 给应用程序需要一个用户态运行环境
	- 进程管理：加载、复制、生命周期、系统调用
	- 内存管理：用户态虚拟内存
- “硬”构造出第一个用户进程
	- 建立用户代码/数据段
	- 创建内核线程
	- 创建用户进程“壳”（创建进程控制块以及其管理的一些关键数据）
	- 填写用户进程“肉”（具体的执行内容）
	- 执行用户进程（在用户空间执行，而不是内核空间）
	- 完成系统调用
	- 结束用户进程

### 进程的内存布局
- 对实际物理空间的映射：0xc0000000到0xf8000000：映射为物理空间（相差0xc0000000的一一映射)
- 页表的建立（当前的page table）：只存在在这一块虚拟空间里面，管理的是整个内核空间里面的映射关系
- 页表和段表的扩展：增加一个用户的代码段和数据段，特权级是3
- 页表内也要管理用户态的空间
- 用户进程的内存空间在一个低端的位置，虽然0xc0000000以下的空间都分给了用户空间，但是用户空间的程序只用了一部分。堆栈，代码段、数据段和它的堆是主要两部分
	- 堆用于动态内存分配
	- 写应用程序的时候可以充分利用这两部分完成函数调用、数据访问、计算等等
- STAB：放调试信息（源码的数据、符号和具体的地址）
- invalid memory：在低地址空间，非法区域。如果应用程序访问了这块区域的地址会产生page fault
	- 设置成invalid是在于应用程序可能会创建一个地址为0的空指针，而0地址是非法的，访问到这里时会产生page fault（便于查出程序的问题）
	- 通过内核的page table，可以对进程的访问空间做限制，使得进程不能任意访问任意地址空间

### 执行ELF格式的二进制代码
- 操作系统把放在硬盘中的应用程序加载到内存中执行（lab5还做不到）
- lab5:一开始的时候bootloader把kernel和程序都放到内存中，一开始ucore kernel和应用程序放在一起，bootloader把这些东西都加载到memory中，导致在后续的创建进程过程中会创建一个壳，然后直接从内存中把这个program放进来去执行，通过do_execve的内核函数来完成创建用户进程且把相应的这个程序放在中间去执行
- do_execve函数的要完成的主要功能是重点，即怎么去执行一个用户的进程
- 用户进程壳
	- ucore中PCB和TCB共用一个数据结构

### 执行elf格式的二进制代码-do_execve的实现
- 把以前的资源（内存空间）去掉，但是保留PID（壳），然后把自己的内容换进去
- 怎么把空间清空：
	- 首先把页表CR3的页表基址指向bootcr3（内核态里的一个页表），退出mmap，put_pgdir和mm_destroy（把进程内存管理那一块的区域给清空）（把对应页表清空）
	- 然后填入新的内容：在load_icode里完成（完成功能比较多，所以会单独讲解），load_icode会把执行程序给加载到新的壳里面，建立新的内存映射关系，从而可以去完成新的执行。如果load_icode能执行完毕，那么整个do_execve都可以正确返回
- load_icode
	- 前面已经把整个内存管理mm_struct清空，那么首先要创建一个新的memory space，建立好新的页表。留出来空间后结构发生了变化，内存指向了不同的地方
	- 然后填上执行代码的内容，执行代码来自bootloader加载ucore的时候顺便加载进来的，所以只要知道起始地址就可以找到对应代码段和数据段。重点是elf格式的header在什么地方，以及header会进一步找到它的各个段（比如代码段）
	- 找到相应的代码段、数据段之后，根据这个代码段、数据段所设定的地址，设定虚地址建立vma（认为这个进程合法的地址空间）对代码段是可执行，对数据段是可读可写（通过mm_map完成了对合法空间的建立）（注意此时建立完之后还没有建立页表，只是做了标识认定合法）
	- 然后把内容拷贝进去~
	- 准备all zero memory。执行程序里有一个bss段的数据需要清空，那么预先做好清空
	- 建立user stack，设置堆栈空间（不在binary code里，需要自己构造，保证用户程序有效执行各种各样的函数调用关系）
	- 载入新的页表：把页表的起始地址从ucore内核的起始地址换到新的起始地址

### 进程复制
- do_fork函数：memory相关一个参数，stack相关两个参数
- 实现：
	- 创建子进程的PCB
	- 进一步初始化其它很多必须内容，比如kernel stack
	- 复制父进程的内存，代码段和数据段一致（为新进程创建一个新的vma）（页表内容重新设置）
	- copy父进程的trapframe并进行改动，重新设置来完成copy和更新的工作

### 内存管理的copy-on-write机制
