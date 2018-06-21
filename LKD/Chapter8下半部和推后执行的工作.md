## Chapter8下半部和推后执行的工作

### 软中断

#### 实现

软中断是在内核编译期间静态分配的，不能动态的注册或注销。

软中断在内核中的表示如下：

```C
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};
```

kernel/softirq.c中定义了含有该结构体的一个数组：

```c
static struct softirq_action softirq_vec[NR_SOFTIRQS]
```



软中断处理函数的原型为：

```c
void softirq_handler(struct softirq_action *)
```

一个注册的软中断必须在被标记后才会被触发，称之为触发软中断。通常，中断处理程序会在返回前标记它的软中断，使其在稍后被执行。



#### 使用软中断

软中断保留给系统中最严格以及最重要的下半部使用。

1. 分配索引，在编译期间，在<linux/interrupt.h>   中增加一个枚举量来静态地声明软中断。内核中原有的软中断如下所示：

   ```c
   enum
   {
   	HI_SOFTIRQ=0,					//高优先级tasklet
   	TIMER_SOFTIRQ,					//定时器的下半部
   	NET_TX_SOFTIRQ,					//发送网络数据
   	NET_RX_SOFTIRQ,					//接受网络数据
   	BLOCK_SOFTIRQ,					//BLOCK装置
   	BLOCK_IOPOLL_SOFTIRQ,			
   	TASKLET_SOFTIRQ,				//一般优先级tasklet
   	SCHED_SOFTIRQ,					//调度程序
   	HRTIMER_SOFTIRQ,				//高分辨率定时器
   	RCU_SOFTIRQ,    				//RCU锁定
   
   	NR_SOFTIRQS
   };
   ```

2. 在运行时调用open_softirq()注册软中断处理程序，比如：

   ```c
   open_softirq(NET_TX_SOFTIRQ, net_tx_action);
   open_softirq(NET_RX_SOFTIRQ, net_rx_action);
   ```

   第一个参数为软中断索引号，第二个参数为处理函数。软中断处理程序运行的时候，允许响应中断，但它自己**不能休眠**。在一个处理程序运行的时候，当前处理器上的软中断被禁止。但其他处理器仍可以执行别的软中断。

3. 触发软中断,比如：

   ```
   raise_softirq(NET_TX_SOFTIRQ)
   ```

   这会触发NET_TX_SOFTIRQ 软中断，它的处理程序就会在内核下一次执行软中断时投入运行。该函数会在触发一个软中断之前先要禁用中断，触发后再恢复原来的状态。如果中断本来就被禁止了，可以调用raise_softirq_irqoff()，这会带来一些优化效果。

   通常会在中断处理程序中触发软中断。

   

### tasklet

tasklet是利用软中断实现的一种下半部机制。

#### 实现

```c
struct tasklet_struct
{
	struct tasklet_struct *next; //指向链表中下一个tasklet
	unsigned long state;		//tasklet的状态
	atomic_t count;				//引用计数
	void (*func)(unsigned long);	//tasklet的处理函数
	unsigned long data;				//tasklet的处理函数的参数
};
```

#### 使用tasklet

1. 声明taklet

   - 静态声明，可以使用下面两个宏

     ```c
     DECLARE_TASKLET(name, func, data)  //引用计数为0，处于激活状态
     DECLARE_TASKLET_DISABLED(name, func, data) //引用计数为1，处于禁止状态
     ```

   - 动态声明，可以调用下面的函数，动态创建

     ```c
     void tasklet_init(struct tasklet_struct *t,
     			 void (*func)(unsigned long), unsigned long data);
     ```

2. 编写taklet处理程序，因为tasklet是靠软中断实现的，所以tasklet不能睡眠。

3. 调度tasklet

   ```c
   tasklet_schedule(&my_tasklet);
   ```

   ​	

### 工作队列

工作队列可以把工作推后交给一个内核 线程去执行——这个下半部总是会在**进程上下文**中执行。这就允许重新调度和睡眠。

#### 实现

工作队列子系统是一个用于创建内核线程的接口，通过它创建的进程负责执行由内核其他部分排到队列里的任务。

![工作、工作队列和工作者线程之间的关系](D:\笔记\Notes\image\工作、工作队列和工作者线程之间的关系.png)

#### 使用工作队列

1. 创建工作

   - 静态创建

     ```c
     DECLARE_WORK(name, void (*func)(void *), void *data);
     ```

   - 动态创建

     ```c
     INIT_WORK(struct work_struct *work, void (*func)(void *), void *data);
     ```

2. 工作队列处理函数

   ```c
   void work_handler(void *data)
   ```

3. 调度工作

   ```c
   schedule_work(&work);
   schedule_delayed_work(&work, delay); //延时delay指定的时钟节拍之后才执行
   ```



### 在下半部加锁

如果进程上下文和一个下半部共享数据，在访问这些数据之前，需要禁止下半部的处理并得到锁的使用权。

如果中断上下文和一个下半部共享数据，在访问数据之前，需要禁止中断并得到锁的使用权。

任何在工作队列中被共享的数据也需要使用锁机制。

```c
void local_bh_disable();//禁止本地处理器的软中断和tasklet的处理
void local_bh_enable(); //激活本地处理器的软中断和tasklet的处理
```



