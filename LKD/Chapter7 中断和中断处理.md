## Chapter7 中断和中断处理



### 中断

中断在本职是一种特殊的电信号，由硬件设备产生，并送给中断控制器。当中断控制器收到一个中断后，会给处理器发送一个信号，处理器检测到该信号，就会中断当前的工作转而处理中断。此后处理器会通知操作系统有中断产生，操作系统会对中断进行处理。

不同的设备对应的中断不同，每个中断都有唯一的数字标识（IRQ line）。这样，操作系统可以对不同的中断进行不同的处理。

在内核中会调用一个函数来处理特定的中断，该函数叫做中断服务程序（ISR）。每个设备产生的中断都对应一个中断服务程序。**中断随时可能发生，中断服务程序也随时可能执行。** 所以中断服务程序必须快速执行，以保证被中断的代码能够尽快恢复。

可以把对中断的处理分为上半部和下半部。中断服务程序是上半部——接收到中断后立即开始执行，但是有严格的时间限制。能够允许稍后执行的工作可以推迟到下半部中处理。

### 注册中断



```C
int request_irq(unsigned int irq,
				irq_handler_t handler,
				unsigned long flags,
				const char *name,
				void *dev)
```

request_irq()函数可能会睡眠。

```C
void free_irq(unsigned int irq, void *dev)    
```

### 中断上下文

当在执行一个中断处理程序时，内核就处于**中断上下文**中。中断上下文中没有后备进程，所以**中断上下文不可以睡眠**。

中断服务程序打乱了其他的代码。中断上下文具有严格的时间限制。

中断处理程序栈是十分有限的，在2.6的内核中，每个处理器有一个栈，大小为一页。



### 中断处理机制的实现



### 中断控制



```c
/*禁用当前处理器上的中断*/
local_irq_disable();
/*使能当前处理器上的中断*/
local_irq_enable();
```



```C
/*禁用中断并保存之前的中断状态*/
unsigned long flags;
local_irq_save(flags); /* interrupts are now disabled */
/* ... */
/*恢复中断并恢复之前的中断状态*/
local_irq_restore(flags); /* interrupts are restored to their previous state */
```



```c
/*禁用指定的中断线*/
void disable_irq(unsigned int irq);
void disable_irq_nosync(unsigned int irq);
/*使能指定的中断*/
void enable_irq(unsigned int irq);
void synchronize_irq(unsigned int irq);
```

disable_irq()只有在当前正在执行的所有处理程序完成后，才会返回。

disable_irq_nosync()不会等待当前中断处理程序执行完毕。

synchronize_irq()等待一个特定的中断处理程序的退出。

对这些函数的调用是可以嵌套的，对一条指定的中断线而言，对disable_irq()或disable_irq_nosync()的每次调用，都需要相应地调用一次enable_irq()，只有在对enable_irq()完成最后一次调用，才真正重新激活了中断线。

这些函数都可以在中断或进程上下文中调用，而且不会睡眠。



```
in_interrupt()
in_irq()
```

如果内核处于任何类型的中断处理中（包括中断服务程序和下半部的处理），in_interrupt()会返回非0。in_irq()函数只有内核确实正在执行中断处理程序时才会返回非0。

