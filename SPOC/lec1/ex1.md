# SPOC_lec1

##第一题
请分析em.c，并补充cpu.md中描述不够或错误的地方。包括：

### 在v9-cpu中如何实现时钟中断
timer是时钟，delta为timer每次增加的量。其中delta为4096，阈值timeout可由TIME指令设置。当timeout不为0且timer大于阈值timeout时，若中断可使能（即iena为1），触发时钟中断，设置中断类型为FTIMER，timer清零，执行中断处理例程。

### v9-cpu指令，关键变量描述有误或不全的情况
- ssp: 系统栈指针。
- usp: 用户栈指针。
- cycle: 指令周期。
- xcycle: 指令周期的四倍。
- timer: 时钟。
- timeout: 时钟阈值，可由TIME指令设置。
- delta: 时钟偏移量，固定为4096。

### 在v9-cpu中的跳转相关操作是如何实现的
- JMP和JMPI指令：直接在XPC上加上偏移量后跳转。
- JSR和JSRA指令：保存XPC的值后跳转。
- BRANCH指令：条件判断，若条件为真则跳转。

### 在v9-cpu中如何设计相应指令，可有效实现函数调用与返回
将参数压入栈中，函数调用开头保存寄存器和sp的值，返回时还原。返回值保存在寄存器a当中。

### emhello/os0/os1等程序被加载到内存的哪个位置,其堆栈是如何设置的
加载到内存的底部。堆栈是从高地址向低地址增长，栈大小为内存大小减去程序大小。

### 在v9-cpu中如何完成一次内存地址的读写
先查看该地址是否存在于TLB当中，若存在则使用，若不存在则将地址写入TLB中，通过页表推出物理地址读写。

### 在v9-cpu中如何实现分页机制
32位地址，前10位指定页表，中间10位指定其中一项，后12位确定页内偏移量。
