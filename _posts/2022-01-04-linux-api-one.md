---
title: 篇一：在Linux内核驱动中常用到API接口函数
layout: post
categories: Linux
tags: linux
excerpt: 在Linux内核中，我们时常会用到API接口函数。
---

## 并发与竞争
## 原子操作
## 定义：
```sh
typedef struct {
    int counter;
} atomic_t;
```

整形操作：
```sh
ATOMIC_INIT(int i);//定义时初始化
int atomic_read(atomic_t *v);
void atomic_set(atomic_t *v, int i)
void atomic_add(int i, atomic_t *v)
void atomic_sub(int i, atomic_t *v)
void atomic_dec(atomic_t *v)//自减
void atomic_inc(atomic_t *v)//自增
int atomic_dec_return(atomic_t *v)//自减并返回v
int atomic_inc_return(atomic_t *v)//自增并返回v
int atomic_sub_and_test(int i, atomic_t *v)//(v-i)==0?1:0（返回真、假）
int atomic_dec_and_test(atomic_t *v)//(v--)==0?1:0（返回真、假）
int atomic_inc_and_test(atomic_t *v)//(v++)==0?1:0（返回真、假）
int atomic_add_negative(int i, atomic_t *v)//(v+i)<0?1:0（返回真、假）
```

位操作（直接对内存操作）：
```sh
void set_bit(int nr, void *p)//将p地址的第nr位置 1
void clear_bit(int nr,void *p)
void change_bit(int nr, void *p)//翻转
int test_bit(int nr, void *p)//获取
int test_and_set_bit(int nr, void *p)//将p地址的第nr位置1，并返回nr位原来的值
int test_and_clear_bit(int nr, void *p)//...清零...
int test_and_change_bit(int nr, void *p)//...翻转...
```

## 自旋锁
基本API

DEFINE_SPINLOCK(spinlock_t lock)//定义并初始化
int spin_lock_init(spinlock_t *lock)//初始化
void spin_lock(spinlock_t *lock)//加锁
void spin_unlock(spinlock_t *lock)//解锁
int spin_trylock(spinlock_t *lock)//尝试加锁
int spin_is_locked(spinlock_t *lock)//检查是否加锁，是返回非 0，否返回 0

## 中断相关：

一般在线程中使用 spin_lock_irqsave/spin_unlock_irqrestore，在中断中使用spin_lock/spin_unlock
```sh
void spin_lock_irq(spinlock_t *lock)//禁止本地中断，并获取自旋锁
void spin_unlock_irq(spinlock_t *lock)//激活本地中断，并释放自旋锁
void spin_lock_irqsave(spinlock_t *lock, unsigned long flags)//保存中断状态，禁止本地中断，并获取自旋锁
void spin_unlock_irqrestore(spinlock_t *lock, unsigned long flags)//恢复中断状态，并且激活本地中断，释放自旋锁
```

中断下半部相关：
```sh
void spin_lock_bh(spinlock_t *lock)//关闭下半部，并获取自旋锁
void spin_unlock_bh(spinlock_t *lock)//打开下半部，并释放自旋锁
```

## 信号量
```sh
DEFINE_SEAMPHORE(name)//定义信号量 并设值为1
void sema_init(struct semaphore *sem, int val)//初始化，并设值为val
void down(struct semaphore *sem)//获取信号量（会导致休眠，不能在中断中使用，不能被信号打断）
int down_interruptible(struct semaphore *sem)//获取信号量(会导致休眠，不能在中断中使用，可以被信号打断)
int down_trylock(struct semaphore *sem)//尝试获取信号量(成功返回 0。否则返回非 0)
void up(struct semaphore *sem)//释放信号量
```

## 互斥体
会导致休眠，不能在中断中使用 mutex，中断中只能使用自旋锁
```sh
DEFINE_MUTEX(name)//定义并初始化
void mutex_init(mutex *lock)//初始化
void mutex_lock(struct mutex *lock)//上锁，失败则休眠，不可以被信号打断
int mutex_lock_interruptible(struct mutex *lock)//上锁，失败则休眠，可以被信号打断
void mutex_unlock(struct mutex *lock)//解锁
int mutex_trylock(struct mutex *lock)//尝试加锁，成功返回1，失败返回0
int mutex_is_locked(struct mutex *lock)//判断是否加锁，是返回1，否返回0
```

## 字符设备驱动
常用宏
```sh
module_init(xxx_init);
module_exit(xxx_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("author");
//printk
#define KERN_EMERG KERN_SOH "0" /* 紧急事件，一般是内核崩溃 */
#define KERN_ALERT KERN_SOH "1" /* 必须立即采取行动 */
#define KERN_CRIT KERN_SOH "2" /* 临界条件，比如严重的软件或硬件错误*/
#define KERN_ERR KERN_SOH "3" /* 错误状态，一般设备驱动程序中使用KERN_ERR 报告硬件错误 */
#define KERN_WARNING KERN_SOH "4" /* 警告信息，不会对系统造成严重影响 */
#define KERN_NOTICE KERN_SOH "5" /* 有必要进行提示的一些信息 */
#define KERN_INFO KERN_SOH "6" /* 提示性的信息 */
#define KERN_DEBUG KERN_SOH "7" /* 调试信息 */
```

## 注册与注销
老方法，不推荐：
```sh
//注册字符设备（浪费设备号）
static inline int register_chrdev(unsigned int major, const char *name, const struct file_operations *fops);
//注销字符设备
static inline void unregister_chrdev(unsigned int major, const char *name);
name：设备名字，指向一串字符串
```
新方法，推荐：
```sh
/*注册（方法一，自己确定设备号）*/
int register_chrdev_region(dev_t from, unsigned count, const char *name);//详见设备号API
cdev.owner = THIS_MODULE;
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
/*注册（方法二，系统分配设备号）*/
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);//详见设备号API
cdev.owner = THIS_MODULE;
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
/*注销*/
void cdev_del(struct cdev *p);
void unregister_chrdev_region(dev_t from, unsigned count);
```

## 设备节点
类相关：
```sh
class_create(owner, name)//创建类 owner一般为THIS_MODULE
void class_destroy(struct class *cls);//卸载类
```

设备相关：
```sh
struct device *device_create(struct class *class, struct device *parent, dev_t devt, void *drvdata, const char *fmt, ...);//创建设备 parent=NULL drvdata=NULL
void device_destroy(struct class *class, dev_t devt)//卸载设备
```

## 设备号
相关宏：
```sh
MAJOR(dev)//dev_t -> 主设备号
MINOR(dev)//dev_t -> 次设备号
MKDEV(ma,mi)//主设备号+次设备号 -> dev_t
```

## 动态分配设备号：

系统分配设备号+自动注册设备号
```sh
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)
dev：存放起始设备编号的指针，当注册成功，*dev就会等于分配到的起始设备编号，可以通过MAJOR()和MINNOR()宏来提取主次设备号
baseminor：次设备号基地址，也就是起始次设备号
count：要申请的数量，一般都是一个
name：字符设备名称
返回值小于0,表示注册失败
```

注册设备号：

手动确定设备号+手动注册设备号
```sh
int register_chrdev_region(dev_t from, unsigned count, const char *name)
from：要申请的起始设备号，也就是给定的设备号
count：要申请的数量，一般都是一个
name：设备名字
当返回值小于0,表示注册失败
```

## 用户空间与内核空间

## 内核空间–>用户空间
```sh
static inline long copy_to_user(void __user *to, const void *from, unsigned long n)
to：用户空间指针
from：内核空间指针
n：从内核空间向用户空间拷贝数据的字节数
成功返回0，失败返回失败数目
```
## 用户空间–>内核空间
```sh
static inline long copy_from_user(void *to, const void __user * from, unsigned long n)
to：内核空间指针
from：用户空间指针
n：从用户空间向内核空间拷贝数据的字节数
成功返回0，失败返回失败数目
```

## 地址映射
```sh
#define ioremap(cookie,size) __arm_ioremap((cookie), (size), MT_DEVICE)
phys_addr：要映射的物理起始地址。
size：要映射的内存空间大小。
void iounmap (volatile void __iomem *addr)
```

## 对映射后的内存进行读写操作的函数：

//读
```sh
u8 readb(const volatile void __iomem *addr)
u16 readw(const volatile void __iomem *addr)
u32 readl(const volatile void __iomem *addr)
```

//写
```sh
void writeb(u8 value, volatile void __iomem *addr)
void writew(u16 value, volatile void __iomem *addr)
void writel(u32 value, volatile void __iomem *addr)
```
## Linux常用命令
```sh
#查看设备号
cat /proc/devices
#创建设备节点
mknod /dev/xxx c 主设备号 次设备号
#查看设备树节点
ls /proc/device-tree
ls /sys/firmware/devicetree/base
#查看platform相关
ls /sys/bus/platform/devices # 设备
ls /sys/bus/platform/drivers # 驱动
#查看misc相关
ls /sys/class/misc # 驱动
#驱动相关
depmod
modprobe xxx.ko
insmod xxx.ko
lsmod
rmmod xxx.ko
```

## 内核定时器
```sh
extern u64 __jiffy_data jiffies_64;
extern unsigned long volatile __jiffy_data jiffies;
```
相关变量：

HZ：宏，就是CONFIG_HZ，系统频率，通过menuconfig配置，默认100
jiffies、jiffies_64：系统运行的节拍数，由宏HZ确定，默认10ms累加一次
jiffies/HZ：系统运行时间（s）
时间先后函数：

//unkown 通常为 jiffies，known 通常是需要对比的值
```sh
time_after(unkown, known)//unkown 时间上超过 known 则返回真
time_before(unkown, known)//unkown 时间上滞后 known 则返回真
time_after_eq(unkown, known)//unkown 时间上超过或等于 known 则返回真
time_before_eq(unkown, known)//unkown 时间上滞后或等于 known 则返回真
```

时间转换函数：
```sh
int jiffies_to_msecs(const unsigned long j)//jiffies类型j --> ms
int jiffies_to_usecs(const unsigned long j)//jiffies类型j --> us
u64 jiffies_to_nsecs(const unsigned long j)//jiffies类型j --> ns
long msecs_to_jiffies(const unsigned int m)//ms --> jiffies
long usecs_to_jiffies(const unsigned int u)//us --> jiffies
unsigned long nsecs_to_jiffies(u64 n)//ns --> jiffies
```
短延时函数：
```sh
void ndelay(unsigned long nsecs)
void udelay(unsigned long usecs)
void mdelay(unsigned long mseces)
```
定时器类型：
```sh
struct timer_list {
    struct list_head entry;
    unsigned long expires; /* 定时器超时时间，单位是节拍数 */
    struct tvec_base *base;
    void (*function)(unsigned long); /* 定时处理函数 */
    unsigned long data; /* 要传递给 function 函数的参数 */
    int slack;
};
```
定时器函数：
```sh
void init_timer(struct timer_list *timer);//初始化
void add_timer(struct timer_list *timer);//注册并使能
int del_timer(struct timer_list * timer);//删除（需等待 定时处理函数 退出）
int del_timer_sync(struct timer_list *timer);//删除（需等待其他处理器使用完定时器，中断上下文勿用）
int mod_timer(struct timer_list *timer, unsigned long expires);//修改定时值（会使能定时器）
返回值： 0，定时器未被使能； 1，定时器已经使能
```

## 中断
## 上半部

申请
```sh
int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev)
irq：要申请中断的中断号。
handler：中断处理函数，当中断发生以后就会执行此中断处理函数。
flags：中断标志，可以在文件include/linux/interrupt.h里面查看所有的中断标志。
```
常用的中断标志：

标志	描述
```sh
IRQF_SHARED	多个设备共享一个中断线，共享的所有中断都必须指定此标志。 如果使用共享中断的话，request_irq函数的 dev 参数是唯一区分他们的标志
IRQF_ONESHOT	单次中断，中断执行一次就结束
IRQF_TRIGGER_NONE	无触发
IRQF_TRIGGER_RISING	上升沿触发
IRQF_TRIGGER_FALLING	下降沿触发
IRQF_TRIGGER_HIGH	高电平触发
IRQF_TRIGGER_LOW	低电平触发
name：中断名字，设置以后可以在/proc/interrupts文件中看到对应的中断名字
dev： 如果将 flags 设置为 IRQF_SHARED 的话， dev 用来区分不同的中断，一般情况下将dev 设置为设备结构体， dev 会传递给中断处理函数 irq_handler_t 的第二个参数
返回值： 0 中断申请成功；其他负值：中断申请失败；-EBUSY：中断已经被申请了
```

对于一种不太重要而又比较耗时的中断，可以使用以下函数进行申请：
```sh
int devm_request_threaded_irq(struct device *dev, unsigned int irq, irq_handler_t handler, irq_handler_t thread_fn, unsigned long irqflags, const char *devname, void *dev_id)
```
该函数会使得中断线程化（中断使用下半部虽然可以被延迟处理，但是依旧先于线程执行，中断线程化可以让这些比较耗时的下半部与进程进行公平竞争）
带devm_前缀表明申请到的资源可以由系统自动释放，无需手动处理
注意：并不是所有的中断都可以被线程化。该函数有利有弊，具体是否使用需要根据实际情况来衡量（触摸中断一般会用）
释放
```sh
void free_irq(unsigned int irq, void *dev)
irq： 要释放的中断
dev：如果中断设置为共享(IRQF_SHARED)的话，此参数用来区分具体的中断。共享中断只有在释放最后中断处理函数的时候才会被禁止掉
```
中断处理函数
格式：
```sh
irqreturn_t (*irq_handler_t) (int, void *)
```

返回值：

一般为return IRQ_RETVAL(IRQ_HANDLED)
```sh
enum irqreturn {
    IRQ_NONE = (0 << 0),
    IRQ_HANDLED = (1 << 0),
    IRQ_WAKE_THREAD = (1 << 1),
};
typedef enum irqreturn irqreturn_t;
```

使能与禁止
```sh
void enable_irq(unsigned int irq)
void disable_irq(unsigned int irq)//需等待 正在执行的中断处理函数执行完
void disable_irq_nosync(unsigned int irq)//无需等待 正在执行的中断处理函数执行完
```

//不推荐
```sh
local_irq_disable()//关闭全局中断
local_irq_enable()//使能全局中断
```

//推荐
```sh
local_irq_save(flags)//保存中断标志到flags并 关闭全局中断
local_irq_restore(flags)//根据flags设置中断标志并 使能全局中断
```

下半部
推荐使用tasklet实现

软中断
定义：
```sh
struct softirq_action
{
    void (*action)(struct softirq_action *);
};
```

系统定义的10个软中断：

多核处理器中，各个 CPU 都有自己的触发和控制机制，并且只执行自己所触发的软中断，但是所执行的软中断服务函数却是相同的：
```sh
enum
{
    HI_SOFTIRQ=0,//高优先级软中断
    TIMER_SOFTIRQ,//定时器软中断
    NET_TX_SOFTIRQ,//网络数据发送软中断
    NET_RX_SOFTIRQ,//网络数据接收软中断
    BLOCK_SOFTIRQ,
    BLOCK_IOPOLL_SOFTIRQ,
    TASKLET_SOFTIRQ,//tasklet 软中断
    SCHED_SOFTIRQ,//调度软中断
    HRTIMER_SOFTIRQ,//高精度定时器软中断
    RCU_SOFTIRQ,//RCU 软中断
    NR_SOFTIRQS
};

static struct softirq_action softirq_vec[NR_SOFTIRQS];
```

注册软中断：

软中断必须在编译的时候静态注册。Linux内核默认会在软中断初始化函数softirq_init中打开TASKLET_SOFTIRQ和HI_SOFTIRQ软中断
```sh
void open_softirq(int nr, void (*action)(struct softirq_action *))
nr：要开启的软中断，在上述enum中选择一个
action：软中断对应的处理函数
```

触发软中断：
```sh
void raise_softirq(unsigned int nr)
```

tasklet
定义：
```sh
struct tasklet_struct
{
    struct tasklet_struct *next; /* 下一个 tasklet */
    unsigned long state; /* tasklet 状态 */
    atomic_t count; /* 计数器，记录对 tasklet 的引用数 */
    void (*func)(unsigned long); /* tasklet 执行的函数，用户定义，相当于中断处理函数 */
    unsigned long data; /* 函数 func 的参数 */
};
```
初始化：
```sh
void tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long data);//初始化
DECLARE_TASKLET(name, func, data)//宏，定义并初始化
func：tasklet 的处理函数
data：要传递给 func 函数的参数
```

调度函数：

一般在中断的上半部中调用
```sh
void tasklet_schedule(struct tasklet_struct *t)
t：要调度的 tasklet，也就是 DECLARE_TASKLET 宏里面的 name
```

工作队列
工作队列在进程上下文执行，即，将要推后的工作交给一个内核线程去执行，因此工作队列允许睡眠或重新调度。

如果你要推后的工作可以睡眠那么就可以选择工作队列，否则的话就只能选择软中断或 tasklet

定义：

工作–>工作队列–>工作者线程

//工作（重点关注）：
```sh
struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func; /* 工作队列处理函数 */
};
```
//工作队列：
```sh
struct workqueue_struct {
    struct list_head pwqs;
    struct list_head list;
    struct mutex mutex;
    int work_color;
    int flush_color;
    atomic_t nr_pwqs_to_flush;
    struct wq_flusher *first_flusher;
    struct list_head flusher_queue;
    struct list_head flusher_overflow;
    struct list_head maydays;
    struct worker *rescuer;
    int nr_drainers;
    int saved_max_active;
    struct workqueue_attrs *unbound_attrs;
    struct pool_workqueue *dfl_pwq;
    char name[WQ_NAME_LEN];
    struct rcu_head rcu;
    unsigned int flags ____cacheline_aligned;
    struct pool_workqueue __percpu *cpu_pwqs;
    struct pool_workqueue __rcu *numa_pwq_tbl[];
};
```
//工作者线程
```sh
struct worker {
    union {
        struct list_head entry;
        struct hlist_node hentry;
    };
    struct work_struct *current_work;
    work_func_t current_func;
    struct pool_workqueue *current_pwq;
    bool desc_valid;
    struct list_head scheduled;
    struct task_struct *task;
    struct worker_pool *pool;
    struct list_head node;
    unsigned long last_active;
    unsigned int flags;
    int id;
    char desc[WORKER_DESC_LEN];
    struct workqueue_struct *rescue_wq;
};
```
初始化：
```sh
#define INIT_WORK(_work, _func)//初始化，需要自己创建work_struct
#define DECLARE_WORK(n, f)//创建和初始化，无需自己创建work_struct
```
调度：

一般在中断的上半部中调用
```sh
bool schedule_work(struct work_struct *work)
work： 要调度的工作
返回值： 0 成功，其他值 失败
```
篇幅较长，拆分成多篇
