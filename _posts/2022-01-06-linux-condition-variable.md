---
title: Linux条件变量
layout: post
categories: Linux
tags: linux
excerpt: 在Linux内核中，条件变量是一种同步机制，允许线程挂起，直到共享数据上的某些条件得到满足。
---

## Linux条件变量
## 相关函数
```sh
#include <pthread.h>
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int pthread_cond_init(pthread_cond_t *cond, pthread_condattr_t *cond_attr);
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex, const struct timespec *abstime);
int pthread_cond_destroy(pthread_cond_t *cond);
```
## 具体说明
条件变量是一种同步机制，允许线程挂起，直到共享数据上的某些条件得到满足。

条件变量上的基本操作有：

触发条件(当条件变为 true 时)
等待条件，挂起线程直到其他线程触发条件
条件变量要和互斥量相联结，以避免出现条件竞争：一个线程预备等待一个条件变量，当它在真正进入等待之前，另一个线程恰好触发了该条件

上述API具体说明如下：
```sh
pthread_cond_init 使用 cond_attr 指定的属性初始化条件变量 cond，当 cond_attr 为 NULL 时，使用缺省的属性
LinuxThreads 实现条件变量不支持属性，因此 cond_attr 参数实际被忽略
pthread_cond_t 类型的变量也可以用 PTHREAD_COND_INITIALIZER 常量进行静态初始化
pthread_cond_signal 使在条件变量上等待的线程中的一个线程重新开始。如果没有等待的线程，则什么也不做。如果有多个线程在等待该条件，只有一个能重启动，但不能指定哪一个。
pthread_cond_broadcast 重新启动等待该条件变量的所有线程。如果没有等待的线程，则什么也不做。
pthread_cond_wait 自动解锁互斥量(如同执行了 pthread_unlock_mutex)，并等待条件变量触发。这时线程挂起，不占用 CPU 时间，直到条件变量被触发。在调用 pthread_cond_wait 之前，应用程序必须加锁互斥量。pthread_cond_wait 函数返回前，自动重新对互斥量加锁(如同执行了 pthread_lock_mutex)。
```
互斥量的解锁和在条件变量上挂起都是自动进行的。因此，在条件变量被触发前，线程需要对互斥量加锁
这种机制可保证在线程加锁互斥量和进入等待条件变量期间，条件变量不被触发
pthread_cond_timedwait 和 pthread_cond_wait 一样，自动解锁互斥量及等待条件变量，但它还限定了等待时间。如果在 abstime 指定的时间内 cond 未触发，互斥量 mutex 被重新加锁，且 pthread_cond_timedwait 返回错误 ETIMEDOUT
abstime 参数指定一个绝对时间，时间原点与 time 和 gettimeofday 相同：abstime = 0 表示 1970 年 1 月 1 日 00:00:00 GMT
pthread_cond_destroy 销毁一个条件变量，释放它拥有的资源。进入 pthread_cond_destroy 之前，必须没有在该条件变量上等待的线程
在 LinuxThreads 的实现中，条件变量不联结资源，除检查有没有等待的线程外，pthread_cond_destroy 实际上什么也不做

## 取消
```sh
pthread_cond_wait 和 pthread_cond_timedwait 是取消点。如果一个线程在这些函数上挂起时被取消，线程立即继续执行，然后再次对 pthread_cond_wait 和 pthread_cond_timedwait 的 mutex 参数加锁，最后执行取消。因此，当调用清除处理程序时，可确保，mutex 是加锁的
```

异步信号安全(Async-signal Safety)
条件变量函数不是异步信号安全的，不应当在信号处理程序中进行调用

特别要注意，如果在信号处理程序中调用 pthread_cond_signal 或 pthread_cond_boardcast 函数，可能导致调用线程死锁。

返回值
在执行成功时，所有条件变量函数都返回 0，错误时返回非零的错误代码

错误代码
```sh
pthread_cond_init、pthread_cond_signal、pthread_cond_broadcast 和 pthread_cond_wait 从不返回错误代码。
pthread_cond_timedwait 函数出错时返回下列错误代码：

ETIMEDOUT：abstime 指定的时间超时时，条件变量未触发
EINTR：pthread_cond_timedwait 被触发中断
```

pthread_cond_destroy 函数出错时返回下列错误代码：
```sh
EBUSY：某些线程正在等待该条件变量
```

## 举例
设有两个共享的变量 x 和 y，通过互斥量 mut 保护，当 x > y 时，条件变量 cond 被触发。
```sh
int x,y;
int x,y;
pthread_mutex_t mut = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
等待直到 x > y 的执行流程：

pthread_mutex_lock(&mut);
while (x <= y) {
    pthread_cond_wait(&cond, &mut);
}
/* 对 x、y 进行操作 */
pthread_mutex_unlock(&mut);
对 x 和 y 的修改可能导致 x > y，应当触发条件变量：

pthread_mutex_lock(&mut);
/* 修改 x、y */
if (x > y) pthread_cond_broadcast(&cond);
pthread_mutex_unlock(&mut);
```

如果能够确定最多只有一个等待线程需要被唤醒(例如，如果只有两个线程通过 x、y 通信)，则使用 pthread_cond_signal 比 pthread_cond_broadcast 效率稍高一些。如果不能确定，应当用pthread_cond_broadcast
要等待在 5 秒内 x > y，则需要这样处理：
```sh
struct timeval now;
struct timespec timeout;
int retcode;

pthread_mutex_lock(&mut);
gettimeofday(&now);
timeout.tv_sec = now.tv_sec + 5;
timeout.tv_nsec = now.tv_usec * 1000;
retcode = 0;
while (x <= y && retcode != ETIMEDOUT) {
    retcode = pthread_cond_timedwait(&cond, &mut, &timeout);
}
if (retcode == ETIMEDOUT) {
    /* 发生超时 */
} else {
    /* 操作 x 和  y */
}
pthread_mutex_unlock(&mut);
```