### 内核同步方法

#### 中断屏蔽

```c
unsigned long flags;
local_irq_save(flags);
/* do something in critical section*/
local_irq_restore(flags);
```

local_irq_disable()和 local_irq_enable()都只能禁止和使能本 CPU 内的中断，因此，并不能解决 SMP 多 CPU 引发的竞态。因此，单独使用中断屏蔽通常不是一种值得推荐的避免竞态的方法， 它适宜与自旋锁联合使用。

#### 原子操作

原子整数操作

```c
void atomic_set(atomic_t *v, int i); /* 设置原子变量的值为 i */
atomic_t v = ATOMIC_INIT(0); /* 定义原子变量 v 并初始化为 0 */
atomic_read(atomic_t *v); /* 返回原子变量的值*/
void atomic_add(int i, atomic_t *v); /* 原子变量增加 i */
void atomic_sub(int i, atomic_t *v); /* 原子变量减少 i */
void atomic_inc(atomic_t *v); /* 原子变量增加 1 */
void atomic_dec(atomic_t *v); /* 原子变量减少 1 */
/*操作并测试*/
int atomic_inc_and_test(atomic_t *v);
int atomic_dec_and_test(atomic_t *v);
int atomic_sub_and_test(int i, atomic_t *v);
/*操作并返回*/
int atomic_add_return(int i, atomic_t *v);
int atomic_sub_return(int i, atomic_t *v);
int atomic_inc_return(atomic_t *v);
int atomic_dec_return(atomic_t *v);
```

此外还有原子64位操作，原子位操作。

#### 自旋锁

为了获得一个自旋锁， 在某 CPU 上运行的代码需先执行一个原子操作，该操作测试并设置（ test-and-set） 某个内存变量，由于它是原子操作，所以在该操作完成之前其他执行单元不可能访问这个内存变量。如果测试结果表明锁已经空闲，则程序获得这个自旋锁并继续执行； 如果测试结果表明锁仍被占用，程序将在一个小的循环内重复这个“ 测试并设置” 操作，即进行所谓的“ 自旋””。 当自旋锁的持有者通过重置该变量释放这个自旋锁后，某个等待的“测试并设置” 操作向其调用者报告锁已释放。

```c
spinlock_t lock; /*定义自旋锁*/
spin_lock_init(&lock) /*动态初始化*/
spin_lock(&lock) /*获取锁，获取不到则自旋*/
spin_trylock(&lock) /*尝试获取锁，获取不到直接返回*/
spin_unlock(&lock)/*释放锁*/
```

**自旋锁在中断处理程序中，一定要在获取锁之前，首先禁用本地中断。**

```c
DEFINE_SPINLOCK(lock);/*静态定义自旋锁*/
unsigned long flags;
spin_lock_irqsave(&lock,flags); /*禁止本地中断，保存本地中断状态，再去获取自旋锁*/
spin_lock_irqrestore(&lock,flags);/*释放锁，恢复中断至加锁前的状态*/
```

 

读写自旋锁是一种比自旋锁粒度更小的锁机制，它保留了“自旋”的概念，但是在写操作方面，只能最多有 1个写进程，在读操作方面，同时可以有多个读执行单元。但是写操作只能有一个任务持有，而且此时不能有并发的读操作。

顺序锁

RCU

#### 信号量

信号量（semaphore）是用于保护临界区的一种常用方法，它的使用方式和自旋锁类似。与自旋锁相同，只有得到信号量的进程才能执行临界区代码。但是，与自旋锁不同的是，**当获取不到信号量时，进程不会原地打转而是进入休眠等待状态**。

```c
#include <linux/semaphore.h>
struct semaphore sem;   //定义信号量
void sema_init(struct semaphore *sem, int val); //动态初始化
#define init_MUTEX(sem)  sema_init(sem, 1) //该宏用于初始化一个用于互斥的信号量，它把信号量 sem 的值设置为 1；
#define init_MUTEX_LOCKED(sem)  sema_init(sem, 0) //该宏也用于初始化一个信号量，但它把信号量 sem 的值设置为 0；
void down(struct semaphore * sem);//获得信号量 sem，因为 down()而进入睡眠状态的进程不能被信号打断
int down_interruptible(struct semaphore * sem);  //因为 down_interruptible()而进入睡眠状态的进程能被信号打断，被信号打断该函数返回会返回非0值
int down_trylock(struct semaphore * sem); //该函数尝试获得信号量 sem，如果能够立刻获得，它就获得该信号量并返回 0，否则，返回非 0 值。可以在中断中使用
void up(struct semaphore * sem); //释放信号量sem
 
```

读写信号量与信号量的关系与读写自旋锁和自旋锁的关系类似，读写信号量可能引起进程阻塞，但它可允许 N 个读执行单元同时访问共享资源，而最多只能有 1 个写执行单元。

#### 互斥体

互斥体与信号量类似，但是更加简洁，同时使用场景也有更严格的限制。

```c
struct mutex my_mutex;
mutex_init(&my_mutex);
mutex_lock(&my_mutex);
mutex_lock_interruptible(&my_mutex);
mutex_trylock(&my_mutex);//不可以在中断中使用
mutex_unlock(&my_mutex);
mutex_unlock(&my_mutex);
```

 

#### 完成变量

如果在内核中一个任务要发出信号通知另一个任务发生了某个特定的事件，可以利用完成变量。

```C
struct completion my_completion; //定义完成变量
init_completion(&my_completion); //初始化完成变量
DECLARE_COMPLETION(my_completion); //静态地创建并初始化
void wait_for_completion(struct completion *c); //等待完成变量被唤醒，非中断
void complete(struct completion *c); //唤醒完成变量，只唤醒一个等待进程
void complete_all(struct completion *c)//唤醒所有的等待进程
```

#### 禁止抢占

```C
preempt_disable();//增加抢占计数，禁止抢占
preempt_enable();//减少抢占技术，允许抢占
```

