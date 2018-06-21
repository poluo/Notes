# Linux系统下应用和内核的通信方式

![1527058823994](D:\笔记\Notes\image\应用于内核通信几种方式的比较.png)

## 基于文件系统

### 字符设备

#### 概要

基于文件系统的通信方式基本都是基于读写文件完成的，使用字符设备，也可以创建一个设备文件以供读写，来达到应用和内核通信的目的。

#### 使用

![1527041884040](D:\笔记\Notes\image\字符设备使用流程.png)



### procfs

#### 概要

 proc虚拟文件系统主要用于内核向用户导出信息，通过它可以在 Linux 内核空间和用户空间之间进行通信。在/proc 文件系统中，我们可以将对虚拟文件的读写作为与内核中实体进行通信的一种手段，与普通文件不同的是，这些虚拟文件的内容都是动态创建的。

#### 使用

procfs对一般的内核与用户通信的应用都能实现。尤其是在进程上下文的环境中内核与用户的的通信，只要内核使用了procfs提供的API将自己的参数以特殊文件形式出口到虚拟文件系统/proc目录下，用户空间程序便可以使用系统提供的对文件操作的系统调用来实现对内核数据的读写，完成数据交互的目的。

![1527041845546](D:\笔记\Notes\image\procfs.png)



```c
static inline struct proc_dir_entry *proc_mkdir(const char *name, struct proc_dir_entry *parent);  //创建目录，name为新建的目录名，parent为父目录，若为NULL,则默认为/proc目录

proc_create(name, mode, parent, proc_fops);//创建文件，name为文件名，mode为文件的访问模式，parent为父目录，proc_fops为文件管理的struct file_operations.

static inline void proc_remove(struct proc_dir_entry *de);//删除目录
remove_proc_entry(name, parent);//删除parent目录下name文件
```

相关的使用demo可参考 内核源码/drivers/block/DAC960.c 

#### 缺点与改进

procfs有一个缺陷，如果输出内容大于1个内存页，需要多次读，因此处理起来很难，另外，如果输出太大，速度比较慢，有时会出现一些意想不到的情况，Alexander Viro实现了一套新的功能，于是出现了seq_file，使得内核输出大文件信息更容易，使用seq_file可以弥补procfs导出大量数据的缺陷，更好地导出文件记录。

seq_file假设我们正在创建的虚拟文件是要顺序遍历的一个项目序列，而这些项目正是必须要返回给用户空间的。用户首先需要包含<linux/seq_file.h>头文件，然后需建立四个迭代器对象，分别实现start、next、stop和show，实现seq_operation结构体，将其与file_operation关联起来接口。

相关使用可以参考 内核源码/drivers/char/toshiba.c

### sysfs

#### 概要

> sysfs is a ram-based filesystem initially based on ramfs. It provides a means to export kernel data structures, their attributes, and the linkages between them to userspace.”               
>
>  														--documentation/filesystems/sysfs.txt 

sysfs文件系统是一个处于内存中的虚拟文件系统，它为我们提供了kobject对象层次结构的视图。帮助用户能以一个简单的文件系统的方式来观察系统中各种设备的拓扑结构，借助属性对象，kobject可以以导出文件的方式，将内核变量提供给用户读取或者写入。

#### 使用

sysfs主要用于导出系统模块的内部参数给用户，一般由sysfs出口的参数都是只读形式的。 

1.  向sysfs中添加和删除kobject

   ```c
   int kobject_add(struct kobject *kobj, struct kobject *parent,const char *fmt, ...)；
   struct kobject *kobject_create_and_add(const char *name, struct kobject *parent)；
   void kobject_del(struct kobject *kobj)；
   ```

2. 向sysfs中添加文件

   1. kobject和kset中有个类型为kobj_type的ktype字段，它含有一个default_attrs，是一个attribute数组，这些属性负责将内核数据映射成sysfs中的文件。

      ```c
      struct attribute {
      	const char		*name; //最终出现在sysfs中的文件名
      	umode_t			mode; //文件权限
      };
      ```

   2. default_attrs列出了默认的属性，sysfs_ops则表明了如何使用它们

      ```
      struct sysfs_ops {
      	ssize_t	(*show)(struct kobject *, struct attribute *,char *);//读取sysfs的调用项
      	ssize_t	(*store)(struct kobject *,struct attribute *,const char *, size_t);//写sysfs的调用项
      	const void *(*namespace)(struct kobject *, const struct attribute *);
      };
      ```

   3. 创建和删除新属性

      ```c
      int __must_check sysfs_create_file(struct kobject *kobj,const struct attribute *attr);
      void sysfs_remove_file(struct kobject *kobj, const struct attribute *attr);
      ```

   参考《linux内核设计与实现》17.4

   使用demo可参考内核源码目录下drivers/acpi/bgrt.c和drivers/rtc/rtc-isl1208.c

### debugfs

#### 概要

debugfs，顾名思义，是一种用于**内核调试**的虚拟文件系统，内核开发者通过debugfs和用户空间交换数据，默认情况下，debugfs会被挂载在目录/sys/kernel/debug之下 。

#### 使用

debugfs使用注册流程与porcfs基本类似，只是所调用的接口不同。

```c
struct dentry *debugfs_create_file(const char *name, umode_t mode,
				   struct dentry *parent, void *data,
				   const struct file_operations *fops); //在指定的parent目录下创建名为name的文件，mode为文件的访问模式，data为驱动私有数据，fops为关联的操作文件使用的函数。

struct dentry *debugfs_create_dir(const char *name, struct dentry *parent); //在debugfs文件系统下创建一个目录，name是要创建的目录名，parent指定创建目录的父目录的dentry，如果为NULL，目录将创建在debugfs文件系统的根目。

static inline void debugfs_remove(struct dentry *dentry)//移除一个目录或文件

```

使用demo可参考内核源码目录下/drivers/s390/char/zcore.c文件



## netlink

#### 概述

netlink是一种特殊的**套接字**，是linux提供的用于内核和用户态进程之间的通信方式。但是注意虽然Netlink主要用于用户空间和内核空间的通信，但是也能用于用户空间的两个进程通信。只是进程间通信有其他很多方式，一般不用Netlink。除非需要用到Netlink的广播特性时。

用户空间可使用标准的BSD socket接口 ，内核态需要使用专门的内核API。

Netlink应用场景广泛，**可以用于进程上下文和中断上下文环境**。

Netlink支持全双工、**异步双工**的通信方式，**内核可以主动向用户空间发送数据**。

Netlink **支持多播**，一个Netlink组可以收到内核模块或应用的多播消息，该消息可以被该Netlink 组的任何内核模块或应用接收到，内核里的事件对用户态的通知机制就应用了这一特性，该子系统会发送给对内核事件感兴趣的应用以内核事件。 



#### 使用

![1527044372240](D:\笔记\Notes\image\netlink使用.png)





用户态使用解析

1. 创建socket

   ```c
   #include <linux/netlink.h>
   netlink_socket = socket(AF_NETLINK, socket_type, netlink_family);
   ```

   Netlink is a **datagram-oriented** service. Both SOCK_RAW and SOCK_DGRAM are valid values for socket_type. However, the netlink protocol does not distinguish between datagram and raw sockets. 

   

   netlink_family selects the kernel module or netlink group to communicate with. netlink_family可以使用内核预定义的socket 协议类型，也可以使用自己定义的。

2. Netlink messages consist of a byte stream with one or multiple nlmsghdr headers and associated payload. In multipart messages (multiple nlmsghdr headers with associated payload in one byte stream) the first and all following headers have the NLM_F_MULTI flag set, except for the last header which has the type NLMSG_DONE.  数据流应该通过[NLMAG_*类的宏](http://man7.org/linux/man-pages/man3/netlink.3.html)去访问。

   ```c
   struct nlmsghdr {
       __u32 nlmsg_len;    /* Length of message including header */
       __u16 nlmsg_type;   /* Type of message content */
       __u16 nlmsg_flags;  /* Additional flags */
       __u32 nlmsg_seq;    /* Sequence number */
       __u32 nlmsg_pid;    /* Sender port ID */
   };
   ```
   nlmsg_type can be one of the standard message types: NLMSG_NOOP message is to be ignored, NLMSG_ERROR message signals an error and the payload contains an nlmsgerr structure, NLMSG_DONE message terminates multipart message. 

3. 寻址格式。sockaddr_nl 结构体描述了内核态或者用户态中一个netlink客户端。

   ```c
     struct sockaddr_nl {
             sa_family_t     nl_family;  /* AF_NETLINK */
             unsigned short  nl_pad;     /* Zero */
             pid_t           nl_pid;     /* Port ID */
             __u32           nl_groups;  /* Multicast groups mask */
     };
   ```

    nl_pid is the unicast address of netlink socket.  It's always 0 if the destination is in the kernel. For a user-space process, nl_pid is usually the PID of the process owning the destination socket.However, **nl_pid identifies a netlink socket, not a process.**  If a process owns several netlink sockets, then nl_pid can be equal to the process ID only for at most one socket.

   nl_groups is a bit mask with every bit representing a netlink group number. 

内核态使用

   	1. 创建netlink_socket
   	2. 发送单播消息 netlink_unicast



[Linux Netlink 基本使用](http://blog.jobbole.com/104334/)

[Why and How to Use Netlink Socket](https://www.linuxjournal.com/article/7356)

[netlink(7)](http://man7.org/linux/man-pages/man7/netlink.7.html)

参考内核源码crypto/cryto_user.c

## 共享内存

#### 概述

系统调用mmap通常用来将文件映射到内存，以加快该文件的读写速度。当用mmap操作一个设备文件时，可以将设备上的存储区域映射到进程的地址空间，从而以内存读写的方法直接控制设备。如果我们在内核模块里申请了一段内存区域，也可以利用此方法，将这个内存映射到用户空间，以实现内核空间与用户空间之间的共享内存。 共享内存能够最大限度的降低内核空间和用户空间之间的数据拷贝, 从而大大提高系统的性能，适用于有大量数据交互的应用场景

#### 使用

![1527055596497](D:\笔记\Notes\image\内核和用户共享内存.png)



使用代码可参考 https://blog.csdn.net/yuzaipiaofei/article/details/6718835



## 系统调用

### 概述

系统调用是内核提供给应用程序的接口，应用对底层硬件的操作大部分都是通过调用系统调用来完成的，比如文件操作函数read、write等。

在linux系统中，每个系统调用都有一个系统调用号。系统调用号一旦分配，就不能再有任何变更，否则更能引起编译好的应用程序崩溃。内核中所有已使用的系统调用列表，都存储在sys_call_table中。这个表为每个系统调用指定了唯一的系统调用号。

linux中是通过引发一个异常来促进系统切换到内核态中执行异常处理程序。

内核在执行系统调用时处于进程上下文，因为新的进程可以使用相同的系统调用，所以**必须保证该系统调用是可重入**的。

### 应用场景

系统调用容易且方便，系统调用具有高性能。

### 使用

1. 实现系统调用，明确系统掉用的参数、返回值和错误码，系统调用的接口应力求简洁，参数尽可能的少。此外，还应该对参数进行仔细的检查，应该包括指针是否有效和是否具有合法权限。
2. 在系统调用表的最后加入一个表项，修改arch\arm\kernel\calls.s 和头文件unistd.h。
3. 将实现放在/kernel下一个相关文件中，比如sys.c或与其功能联系最紧密的代码中。

可参考《linux内核设计与实现》第五章相关内容