# 内核数据结构

## 链表

 链表：单向链表、双向链表、环形链表

**在linux内核中，不是将数据结构塞入链表，而是将链表节点塞入数据结构。**

linux内核中链表定义如下

```c
struct list_head {
	struct list_head *next, *prev;
};
```

初始化链表

```c
INIT_LIST_HEAD(&list)//动态初始化
LIST_HEAD_INIT(list)//静态初始化
LIST_HEAD(list_name) //初始化并定义一个名为list_name的头结点
```

操作链表

```c
//将new节点插入head节点之后
void list_add(struct list_head *new, struct list_head *head)
//将new节点插入head节点之前
void list_add_tail(struct list_head *entry, struct list_head *head)
//从链表中删除entry元素，但不会释放entry或包含entry元素的数据结构
void list_del(struct list_head *entry)
//entry从链表删除下来，重新初始化。这样做是因为：虽然不在需要entry项，但是还可以再使用包含entry的数据结构体
void list_del_init(struct list_head *entry)
//从一个链表中移除list项，然后将其加入另一链表的head节点后
void list_move(struct list_head *list, struct list_head *head)
//从一个链表中移除list项，然后将其加入另一链表的head节点前
void list_move_tail(struct list_head *list, struct list_head *head)
//如果链表为空，返回非0，否则返回0
int list_empty(const struct list_head *head)
//合并链表，将list指向链表插入指定链表head节点后
static inline void list_splice(const struct list_head *list, struct list_head *head)
//合并链表，将list指向链表插入指定链表head节点前
static inline void list_splice_tail(struct list_head *list, struct list_head *head)
static inline void list_splice(const struct list_head *list, struct list_head *head)
//与list_splice()函数一样， 不同的是由list指向的链表要被重新初始化
static inline void list_splice_init(struct list_head *list, struct list_head *head) 
//与list_splice_tail一样同的是由list指向的链表要被重新初始化
static inline void list_splice_tail_init(struct list_head *list, struct list_head *head) 
```

遍历链表

```c

//第一个参数用来指向当前项，这是一个你必须提供的临时变量，第二个参数是需要遍历的链表的头节点。每次遍历中，第一个参数在链表中不断移动指向下一个元素，直到链表中的所有元素都被访问为止
#define list_for_each(pos, head) \
    for (pos = (head)->next; pos != (head); pos = pos->next)
//pos指向包含list_head节点对象的指针，可将其看做是list_entry()的返回值。head是一个指向头结点的指针，即遍历开始位置，member是pos中list_head结构的变量名。
#define list_for_each_entry(pos, head, member)              \
    for (pos = list_entry((head)->next, typeof(*pos), member);  \
         &pos->member != (head);    \
         pos = list_entry(pos->member.next, typeof(*pos), member))

//反向遍历链表，与list_for_each_entry()类似，不同点在于它是反向遍历链表。
#define list_for_each_entry_reverse(pos, head, member)			\
	for (pos = list_entry((head)->prev, typeof(*pos), member);	\
	     &pos->member != (head); 	\
	     pos = list_entry(pos->member.prev, typeof(*pos), member))
//遍历链表的同时删除元素，与list_for_each_entry()使用类似，只是需要提供n指针，n指针和pos是通用的类型
#define list_for_each_entry_safe(pos, n, head, member)			\
	for (pos = list_entry((head)->next, typeof(*pos), member),	\
		n = list_entry(pos->member.next, typeof(*pos), member);	\
	     &pos->member != (head); 					\
	     pos = n, n = list_entry(n->member.next, typeof(*n), member))

//类似的还有反向遍历链表的同时删除元素
list_for_each_entry_safe_reverse(pos, n, head, member)
```

更多链表操作方法参见<linux/list.h>

## 队列



```c
//创建队列
int kfifio_alloc(struct kfifo *kfifo, unsigned int size, gfp_t gfp_mask);
//入列
unsigned int kfifo_in(struct kfifo *kfifo, const void *from, unsigned int len);
//出列
unsigned int kfifo_out(struct kfifo *kfifo, const void *to, unsigned int len);
//取出数据，但不删除, offset为队列中索引位置，offset为0即为队列头
unsigned int kfifo_out_peek(struct kfifo *kfifo, const void *to ,unsigned int len, unsigned int offset);
//获取队列空间的总大小
unsigned int kfifo_size(struct kfifo *kfifo);
//返回队列中已推入数据的大小
unsigned int kfifo_len(struct kfifo *kfifo);
//返回对垒中还有多少空间可用
unsigned int kfifo_avail(struct kfifo *fifo);
//判断队列为空
int kfifo_is_empty(struct kfifo *fifo);
//判断队列为满
int kfifo_is_full(struct kfifo *fifo);
//重置队列，抛弃队列中所有内容
void kfifo_reset(struct kfifo *fifo);
//释放对列
void kfifo_free(struct kfifo *fifo);
```



## 映射

一个映射，是一个由唯一键组成的集合，每个键必然关联一个特定的值。映射应至少支持三个操作：

1. Add(value, key)
2. Remove(key)
3. value = Lookup(key)

linux内核中的映射不是一个通用的映射，它的目的是**映射一个唯一的标识数（UID）到一个指针**。



初始化idr

```c
void idr_init(struct idr *idrp);

struct idr idr_huh
idr_init(&idr_huh);
```

分配一个新的UID，需要两步，第一步高速idr你需要分配新的UID，允许其在必要时调整后备树的大小，第二步才是真正请求新的UID。

```c
//调整由idp指向idr的大小，成功返回1，失败返回0
int idr_pre_get(struct idr *idp, gfp_t gpf_mask);
//使用idp指向idr去分配一个新的UID,新的UID存在id中，并将其关联到指针ptr上
int idr_get_new(struct idr *idp, void *ptr, int *id);

//可指定最小的UID返回值，确保新的UID大于或等于string ID，且确保UID是唯一的。
int idr_get_new_above(struct idr *idrp, void *ptr, int string_id, int *id);

//使用示例
int id;
int ret;
do{
	if(!idr_pre_get(&idr_huh, GFP_KERNEL))
	{
        return -ENOSPC;
	}
	ret = idr_get_new(&idr_huh, ptr, &id);
}while(ret == -EAGAIN)
```

```c
//在idr中查找UID,若查找成功返回id相关联的指针，否则返回空指针。
void *idr_find(struct idr *idp, int id);

//从idr中删除UID
void idr_remove(struct idr *idp, int id);

//撤销idr,只释放idr中未使用的内存，并不释放当前分配给UID使用的任何内存
void idr_destory(struct idr *idp);
//强制删除所有UID
void idr_remove_all(struct idr *idp);
```

## 二叉树

### 二叉搜索树

二叉搜索树(binary search trees)，是一个节点有序的二叉树，其遵循下列法则：

- 根的左分支节点值都小于根节点值
- 有分支节点值都大于根节点值
- 所有的子树也都是二叉搜索树

### 自平衡二叉搜索树

所有叶子节点深度差不超过1的二叉搜索树。 红黑树，是一种自平衡二叉搜索树，linux主要的平衡二叉树数据结构就是红黑树。红黑树具有特殊的着色属性，遵循下面六个属性，能维持半平衡结构： 

1. 所有叶子节点要么着红色，要么着黑色。
2. 叶子节点都是黑色。
3. 叶子节点不包含数据。
4. 所有非叶子节点都有两个字节点。
5. 如果一个节点是红色，则它的子节点都是黑色。
6. 在一个节点到其叶子节点的路径中，如果总是包含同样数目的黑色节点，则该路径相比其他路径是最短的。

上述条件保证了最深叶子节点的深度不会大于两倍的最浅叶子节点的深度，所以红黑树总是半平衡的。 

红黑树的实现在<lib/rbtree.c>