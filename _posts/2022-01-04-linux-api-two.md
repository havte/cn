---
title: 篇二：在Linux内核驱动中常用到API接口函数
layout: post
categories: Linux
tags: linux
excerpt: 在Linux内核中，我们时常会用到API接口函数。阻塞与非阻塞、异步通知注解等
---

## 阻塞与非阻塞
阻塞：访问资源时，如果资源不可用，将会挂起线程，直到资源可用再唤醒。open函数中大部分flag均为阻塞访问

非阻塞：访问资源时，如果资源不可用，将会直接返回错误码。open函数中flag参数O_NONBLOCK为非阻塞访问

## 等待队列
一般用于在中断中唤醒阻塞操作而引起的线程挂起

等待队列头
等待队列的头部

定义：
```sh
struct __wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
```
初始化：
```sh
void init_waitqueue_head(wait_queue_head_t *q)//仅初始化
DECLARE_WAIT_QUEUE_HEAD//宏，定义+初始化
```
## 等待队列项
每个访问设备的进程都需要创建一个队列项，当设备不可用的时候需要将这些进程对应的等待队列项添加到等待队列里面

定义：
```sh
struct __wait_queue {
    unsigned int flags;
    void *private;
    wait_queue_func_t func;
    struct list_head task_list;
};
typedef struct __wait_queue wait_queue_t;
```
初始化：
```sh
DECLARE_WAITQUEUE(name, tsk)//宏，定义并初始化
name：等待队列项的名字
tsk：这个等待队列项属于哪个任务(进程)，一般设置为current（全局变量，表示当前进程）
```

## 添加/移除等待队列
当设备不可用的时候需要将该进程对应的等待队列项添加到等待队列里面
```sh
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)//添加
void remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)//删除
q： 等待队列项要加入的等待队列头
wait：要加入/删除的等待队列项
```

## 等待唤醒
当设备可以使用的时候就要唤醒进入休眠态的进程
```sh
void wake_up(wait_queue_head_t *q)//唤醒队列中的所有进程（包括TASK_INTERRUPTIBLE 和 TASK_UNINTERRUPTIBLE状态）
void wake_up_interruptible(wait_queue_head_t *q)//唤醒队列中的所有进程（仅唤醒TASK_INTERRUPTIBLE状态进程）
```

## 等待事件
当事件满足以后能够自动唤醒等待队列中的进程
```sh
wait_event(wq, condition)//若condition为真，则唤醒等待队列，否则一直阻塞。（会将进程设置为TASK_UNINTERRUPTIBLE状态）
wait_event_timeout(wq, condition, timeout)//与wait_event类似，timeout为超时时间，单位为jiffies。返回0：超时时间到，且condition为假；返回1：condition为真
wait_event_interruptible(wq, condition)//与wait_event类似，此函数会将进程设置为TASK_INTERRUPTIBLE，即可以被信号打断
wait_event_interruptible_timeout(wq, condition, timeout)//与wait_event_timeout类似，此函数会将进程设置为TASK_INTERRUPTIBLE，即可以被信号打断
```

## 轮询
一般用于非阻塞操作

poll、 epoll 和 select 用于处理轮询。
应用程序通过 select、 epoll 或 poll 来查询设备是否可以操作，如果可以操作的话就从设备读取或者向设备写入数据。当应用程序调用 select、 epoll 或 poll 函数的时候设备驱动程序中的 poll 函数就会执行

## select
单线程中 select 函数能够监视的文件描述符数量一般最大为1024，可以修改该数量但会降低效率
```sh
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout)
nfds： 要操作的文件描述符个数
readfds：指向描述符集合，fd_set类型。用于监视指定描述符集的读变化，所指定集合中有一个文件可以读取，则返回大于 0 的值，否则根据 timeout 参数来判断是否超时。设置为 NULL，表示不关心任何文件的读变化
writefds：指向描述符集合，fd_set类型。用于监视指定描述符集的写变化，所指定集合中有一个文件可以写入，则返回大于 0 的值，否则根据 timeout 参数来判断是否超时。设置为 NULL，表示不关心任何文件的写变化
exceptfds：指向描述符集合，fd_set类型。用于监视指定描述符集的异常变化，所指定集合中有一个文件异常，则返回大于 0 的值，否则根据 timeout 参数来判断是否超时。设置为 NULL，表示不关心任何文件的异常变化
```

fd_set变量定义相关宏：
```sh
void FD_ZERO(fd_set *set)//将 fd_set 变量的所有位都清零
void FD_SET(int fd, fd_set *set)//将 fd_set 变量的某个位置 1（向 fd_set 添加一个文件描述符）， fd：要加入的文件描述符
void FD_CLR(int fd, fd_set *set)//将 fd_set 变量的某个位清零（向 fd_set 删除一个文件描述符）， fd：要删除的文件描述符
int FD_ISSET(int fd, fd_set *set)//测试 fd_set 的某个位是否置 1（判断某个文件是否可以进行操作）， fd：要判断的文件描述符
timeout：超时时间，设为 NULL 表示无限期等待。timeval结构体类型

struct timeval {
 long tv_sec; /* 秒 */
 long tv_usec; /* 微秒 */
};
返回值： 0，超时发生，没有任何文件描述符可以进行操作； -1，发生错误；其他值，可以进行操作的文件描述符个数
```

select使用示例：
```sh
void main(void)
{
    int ret, fd; /* 要监视的文件描述符 */
    fd_set readfds; /* 读操作文件描述符集 */
    struct timeval timeout; /* 超时结构体 */
    fd = open("dev_xxx", O_RDWR | O_NONBLOCK); /* 非阻塞式访问 */
    FD_ZERO(&readfds); /* 清除 readfds */
    FD_SET(fd, &readfds); /* 将 fd 添加到 readfds 里面 */
    /* 构造超时时间 */
    timeout.tv_sec = 0;
    timeout.tv_usec = 500000; /* 500ms */
    ret = select(fd + 1, &readfds, NULL, NULL, &timeout);
    switch (ret) {
        case 0: /* 超时 */
            ......
            break;
        case -1: /* 错误 */
            ......
            break;
        default: /* 可以读取数据 */
            if(FD_ISSET(fd, &readfds)) { /* 判断是否为 fd 文件描述符 */
                /* 使用 read 函数读取数据 */
            }
            break;
    }
}
```

## poll
本质上和 select 没有太大的差别，但是 poll 函数没有最大文件描述符限制

int poll(struct pollfd *fds, nfds_t nfds, int timeout)
fds： 要监视的文件描述符集合以及要监视的事件，为结构体pollfd数组类型，pollfd类型定义：
```sh
struct pollfd {
    int fd; /* 文件描述符 */
    short events; /* 请求的事件 */
    short revents; /* 返回的事件 */
};
```
fd：要监视的文件描述符，如果 fd 无效则 events 监视事件也无效，并且 revents 返回 0

events：要监视的事件，类型如下：
```sh
POLLIN 有数据可以读取
POLLPRI 有紧急的数据需要读取
POLLOUT 可以写数据
POLLERR 指定的文件描述符发生错误
POLLHUP 指定的文件描述符挂起
POLLNVAL 无效的请求
POLLRDNORM 等同于POLLIN
```

revents：返回参数，由 Linux 内核设置具体的返回事件
nfds：poll 函数要监视的文件描述符数量
timeout：超时时间，单位为 ms
返回值：返回 revents 域中不为 0 的 pollfd 结构体个数，即发生事件或错误的文件描述符数量； 0，超时； -1，发生错误，并设置相应的错误码

poll使用示例：
```sh
void main(void)
{
    int ret;
    int fd; /* 要监视的文件描述符 */
    struct pollfd fds;
    fd = open(filename, O_RDWR | O_NONBLOCK); /* 非阻塞式访问 */
    /* 构造结构体 */
    fds.fd = fd;
    fds.events = POLLIN; /* 监视数据是否可以读取 */
    ret = poll(&fds, 1, 500); /* 轮询文件是否可操作，超时 500ms */
    if (ret) { /* 数据有效 */
        ......
        /* 读取数据 */
        ......
    } else if (ret == 0) { /* 超时 */
        ......
    } else if (ret < 0) { /* 错误 */
        ......
    }
}
```

## epoll
selcet 和 poll 函数会随着所监听的 fd 数量的增加，而导致效率的降低，且遍历所有的描述符比较浪费时间。epoll 专为处理大并发而准备，一般用于网络编程中。 其相关函数如下：

创建 epoll 句柄：

int epoll_create(int size)
size：Linux2.6.8 后此参数已无意义，大于 0 即可
返回值： epoll 句柄，如果为-1 的话表示创建失败
向 epoll 句柄中添加要监视的文件描述符以及监视的事件：
```sh
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
epfd：epoll 句柄（即epoll_create的返回值）
```

op：对 epfd 的操作，可以设置为：
```sh
EPOLL_CTL_ADD 向epfd添加文件参数fd表示的描述符
EPOLL_CTL_MOD 修改参数fd的event事件
EPOLL_CTL_DEL 从epfd中删除fd描述符
```

fd：要监视的文件描述符
event：要监视的事件类型，epoll_event结构体指针类型：
```sh
struct epoll_event {
    uint32_t events; /* epoll 事件 */
    epoll_data_t data; /* 用户数据 */
};
```
events：要监视的事件：
```sh
EPOLLIN 有数据可以读取
EPOLLPRI 有紧急的数据需要读取
EPOLLOUT 可以写数据
EPOLLERR 指定的文件描述符发生错误
EPOLLHUP 指定的文件描述符挂起
EPOLLET 设置epoll为边沿触发（默认为水平触发）
EPOLLONESHOT 一次性监视（当监视完成以后需要再次监视某个fd，就需要将fd重新添加到epoll里面）
```
返回值： 0，成功； -1，失败，并设置相应的错误码

等待事件发生：
```sh
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
epfd：epoll 句柄
events：指向 epoll_event 结构体的数组，当有事件发生的时候 Linux 内核会填写 events，调用者可以根据 events 判断发生了哪些事件
maxevents：events 数组大小（必须大于 0）
timeout：超时时间，单位为 ms
```
返回值： 0，超时； -1，错误；其他值，准备就绪的文件描述符数量

驱动中的poll函数
函数原型：

unsigned int (*poll) (struct file *filp, struct poll_table_struct *wait)
filp： 要打开的设备文件(文件描述符)

wait： 结构体 poll_table_struct 类型指针， 由应用程序传递进来的。一般将此参数传递给
poll_wait 函数

返回值：向应用程序返回设备或者资源状态，可以返回的资源状态如下：
```sh
POLLIN 有数据可以读取
POLLPRI 有紧急的数据需要读取
POLLOUT 可以写数据
POLLERR 指定的文件描述符发生错误
POLLHUP 指定的文件描述符挂起
POLLNVAL 无效的请求
POLLRDNORM 等同于POLLIN，普通数据可读
```

添加到poll_table：

非阻塞函数，不会引起阻塞，只是将应用程序添加到 poll_table 中
```sh
void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
wait_address：要添加到 poll_table 中的等待队列头
p：poll_table，就是file_operations 中 poll 函数的 wait 参数
```

驱动通用模板：
```sh
struct xxxx_dev {
    ......
    struct wait_queue_head_t r_wait; /* 等待队列头 */
};

unsigned int xxx_poll(struct file *filp, struct poll_table_struct *wait)
{
    unsigned int mask = 0;
    struct xxxx_dev *dev = (struct xxxx_dev *)filp->private_data;

    poll_wait(filp, &dev->r_wait, wait);

    if(......) { /* 相关条件满足 */
        mask = POLLIN | POLLRDNORM; /* 返回 PLLIN */
    }
    return mask;
}
```

## 异步通知（信号）
Linux中的信号宏：
```sh
#define SIGHUP 		1 /* 终端挂起或控制进程终止 */
#define SIGINT 		2 /* 终端中断(Ctrl+C 组合键) */
#define SIGQUIT 	3 /* 终端退出(Ctrl+\组合键) */
#define SIGILL 		4 /* 非法指令 */
#define SIGTRAP 	5 /* debug 使用，有断点指令产生 */
#define SIGABRT 	6 /* 由 abort(3)发出的退出指令 */
#define SIGIOT 		6 /* IOT 指令 */
#define SIGBUS 		7 /* 总线错误 */
#define SIGFPE 		8 /* 浮点运算错误 */
#define SIGKILL 	9 /* 杀死、终止进程 */
#define SIGUSR1 	10 /* 用户自定义信号 1 */
#define SIGSEGV 	11 /* 段违例(无效的内存段) */
#define SIGUSR2 	12 /* 用户自定义信号 2 */
#define SIGPIPE 	13 /* 向非读管道写入数据 */
#define SIGALRM 	14 /* 闹钟 */
#define SIGTERM 	15 /* 软件终止 */
#define SIGSTKFLT 	16 /* 栈异常 */
#define SIGCHLD 	17 /* 子进程结束 */
#define SIGCONT 	18 /* 进程继续 */
#define SIGSTOP 	19 /* 停止进程的执行，只是暂停 */
#define SIGTSTP 	20 /* 停止进程的运行(Ctrl+Z 组合键) */
#define SIGTTIN 	21 /* 后台进程需要从终端读取数据 */
#define SIGTTOU 	22 /* 后台进程需要向终端写数据 */
#define SIGURG 		23 /* 有"紧急"数据 */
#define SIGXCPU 	24 /* 超过 CPU 资源限制 */
#define SIGXFSZ 	25 /* 文件大小超额 */
#define SIGVTALRM 	26 /* 虚拟时钟信号 */
#define SIGPROF 	27 /* 时钟信号描述 */
#define SIGWINCH 	28 /* 窗口大小改变 */
#define SIGIO 		29 /* 可以进行输入/输出操作 */
#define SIGPOLL 	SIGIO
/* #define SIGLOS 29 */
#define SIGPWR 		30 /* 断点重启 */
#define SIGSYS 		31 /* 非法的系统调用 */
#define SIGUNUSED 	31 /* 未使用信号 */
```
上述信号中，除了 SIGKILL(9)和 SIGSTOP(19)这两个信号不能被忽略外，其他的信号都可以忽略

应用程序API
指定信号的处理函数：
```sh
sighandler_t signal(int signum, sighandler_t handler)
```
signum：要设置处理函数的信号
handler： 信号的处理函数，函数原型：
```sh
typedef void (*sighandler_t)(int)
```
返回值： 设置成功返回信号的前一个处理函数，设置失败返回 SIG_ERR

应用程序的一般模板：
```sh
fd = open(filename, O_RDWR);
if (fd < 0) {
    printf("Can't open file %s\r\n", filename);
    return -1;
}

/* 设置信号 SIGIO 的处理函数 */
signal(SIGIO, sigio_signal_func);

fcntl(fd, F_SETOWN, getpid()); /* 将当前进程的进程号告诉给内核 */
flags = fcntl(fd, F_GETFD); /* 获取当前的进程状态 */
fcntl(fd, F_SETFL, flags | FASYNC);/* 设置进程状态为 FASYNC 启用异步通知功能（此时会调用驱动中的fasync函数） */

while(1) {
    sleep(2);
}

close(fd);
return 0;
```

## 驱动程序API
异步相关结构体：
```sh
struct fasync_struct {
    spinlock_t fa_lock;
    int magic;
    int fa_fd;
    struct fasync_struct *fa_next;
    struct file *fa_file;
    struct rcu_head fa_rcu;
};
```
向应用程序发送中断信号：
```sh
void kill_fasync(struct fasync_struct **fp, int sig, int band)
```
fp：要操作的 fasync_struct 结构体
sig： 要发送的信号
band： 可读时设置为 POLL_IN，可写时设置为 POLL_OUT
返回值： 无

file_operations操作集中异步接口函数：
应用程序通过fcntl(fd, F_SETFL, flags | FASYNC)改变fasync标记的时候，该函数就会执行
```sh
int (*fasync) (int fd, struct file *filp, int on)
```
初始化 fasync_struct 结构体：
```sh
int fasync_helper(int fd, struct file * filp, int on, struct fasync_struct **fapp)
```
前三个参数与fasync函数中参数一致
fapp：要初始化的 fasync_struct 结构体指针变量

驱动一般模板：
```sh
struct xxx_dev {
    ......
    struct fasync_struct *async_queue; /* 异步相关结构体 */
};

static int xxx_fasync(int fd, struct file *filp, int on)
{
    struct xxx_dev *dev = (xxx_dev)filp->private_data;

    if (fasync_helper(fd, filp, on, &dev->async_queue) < 0)//直接调用该函数即可，不用干其他事
        return -EIO;
    return 0;
}

static struct file_operations xxx_ops = {
    ......
    .fasync = xxx_fasync,
    .release = xxx_release,
    ......
};

static int xxx_release(struct inode *inode, struct file *filp)
{
    return xxx_fasync(-1, filp, 0); /* 删除异步通知 */
}
```
