一个Linux下的超级简洁的定时器：*利用epoll机制和`timerfd`新特性实现的多重、多用、多个定时任务实现*。

https://gitee.com/simpost/mult_timer



把每个函数的参数都要弄清楚。虽然人工智能帮我们把能做的东西简化了，但是必懂得知识必须会。

大致流程如下所示：

- 创建定时器
- 启动定时器
- 获取定时器剩余时间
- 删除定时器



# 一、创建定时器

**头文件包含**

```
#include <signal.h>
#include <time.h>
```



**timer_create函数**

```
int timer_create(clockid_t clockid, struct sigevent *sevp, timer_t *timerid);
```

timerid装载的是被创建的定时器的ID



clockid说明定时器是基于哪个时钟的，取值可以为

CLOCK_REALTIME : 可设置的系统范围的实时时钟（一般传递这个参数）

CLOCK_MONOTONIC: 单调递增的时钟，系统启动后不会被改变

CLOCK_PROCESS_CPUTIME_ID : 当前进程的CPU占用时间，包含用户调用和系统调用，

CLOCK_THREAD_CPUTIME_ID : 当前线程的CPU占用时间，包含用户调用和系统调用

以上是常用的，剩余的感兴趣的可以自己研究下。



**sigevent结构体**

参考文档：https://man7.org/linux/man-pages/man7/sigevent.7.html

```
/* 随通知所传递的数据 */
union sigval { 
   int     sival_int; /*int类型数据*/
   void   *sival_ptr; /*指针数据*/
};

struct sigevent {
   int          sigev_notify; /* 通知方式 */
   int          sigev_signo;  /* 通知信号 */
   union sigval sigev_value;  /* 随通知所传递的数据 */

   /* 通知方式设置为"线程通知" (SIGEV_THREAD)时新线程的线程函数 */
   void       (*sigev_notify_function) (union sigval); 

   /* 通知方式设置为"线程通知" (SIGEV_THREAD)时新线程的属性 */
   void        *sigev_notify_attributes;

   /* 通知方式为SIGEV_THREAD_ID时，接收信号的线程的pid */                        
   pid_t        sigev_notify_thread_id;              
};
```

通知方式sigev_notify字段定义了通知的方式，可取的值有以下几种：

**SIGEV_NONE**：当定时器超时时，不异步通知；

**SIGEV_SIGNAL**：通过发送`sigev_signo`所指定的信号的方式来通知进程。

**SIGEV_THREAD**：当定时器到期，内核会(在此进程内)以sigev_notification_attributes为线程属性创建一个线程，并且让它执行sigev_notify_function，传入sigev_value作为为一个参数。

 **SIGEV_THREAD_ID**（linux特有）：和SIGEV_SIGNAL类似，信号的目标是sigev_notify_thread_id参数所指定的线程，sigev_notify_thread_id指定的是内核线程ID，由clone或gettid函数返回。

**NULL** ： 如果sevp被设置为NULL，相当于SIGEV_SIGNAL，信号是默认的SIGALRM



# 二、启动定时器

timer_create()所创建的定时器并未启动。要将它关联到一个到期时间以及启动时钟周期，可以使用timer_settime()。

**timer_settime函数**

```
int timer_settime(timer_t timerid, int flags, const struct itimerspec *new_value, struct itimerspec * old_value);
```



使用到的时间参数数据结构体如下：

```
struct timespec {
    time_t tv_sec;                /* Seconds */
    long   tv_nsec;               /* Nanoseconds */
};*

struct itimerspec {
    struct timespec it_interval;  /* Timer interval */
    struct timespec it_value;     /* Initial expiration */
};
```

参数解析：

itimerspec将时间分为秒和纳秒的组合；

如果参数new_value->it_value不为0，则启动定时器。

如果参数new_value->it_value为0则关闭定时器。

参数new_value->it_interval，为当前定时器超时后，到下一次超时的时间间隔。it_interval的值不能为0；

参数flags为0的话默认是相对时间，也可以使用TIMER_ABSTIME绝对时间



# 三、返回下一次超时的时间间隔

```
int timer_gettime(timer_t timerid, struct itimerspec *curr_value);
```



# 四、删除定时器

```
int timer_delete(timer_t timerid); 
```



# 五、示例程序

linux定时器的编程实现

```
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <signal.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

void expired(union sigval timer_data);
pid_t gettid(void);

#define EXPIRE_MAX 5
static int expire_cnt;

//事件数据结构体定义
struct t_eventData{
    int myData;
};

int main()
{
    int res = 0;
    timer_t timerId = 0;

    struct t_eventData eventData = { .myData = 0 };


    /*  sigevent specifies behaviour on expiration  */
    struct sigevent sev = { 0 };

    /* specify start delay and interval
     * it_value and it_interval must not be zero */

    struct itimerspec its = {   .it_value.tv_sec  = 1,
                                .it_value.tv_nsec = 0,
                                .it_interval.tv_sec  = 1,
                                .it_interval.tv_nsec = 0
                            };

    printf("Simple Threading Timer - thread-id: %d\n", gettid());

    sev.sigev_notify = SIGEV_THREAD;//线程形式信号通知
    sev.sigev_notify_function = &expired;//设置信号通知函数
    sev.sigev_value.sival_ptr = &eventData;


    /* 创建定时器 */
    res = timer_create(CLOCK_REALTIME, &sev, &timerId);
    if (res != 0){
        fprintf(stderr, "Error timer_create: %s\n", strerror(errno));
        exit(-1);
    }

    /* 启动定时器 */
    res = timer_settime(timerId, 0, &its, NULL);
    if (res != 0){
        fprintf(stderr, "Error timer_settime: %s\n", strerror(errno));
        exit(-1);
    }
    
    /* 超时5次后退出 */
    while (expire_cnt != 5) 
    {
        sleep(1);
    }
	
    /* 删除定时器 */
    timer_delete(timerid);
    return 0;
}


void expired(union sigval timer_data){
    struct t_eventData *data = timer_data.sival_ptr;
    printf("Timer fired %d - thread-id: %d\n", ++data->myData, gettid());
    
    if (expire_cnt < EXPIRE_MAX)
    {
    	//走一次超时处理函数，则累加一次
        expire_cnt++;
    }
}

```

其流程分为以下几步：

1、创建一个timer_t类型的定时器变量和一个sigevent类型的事件变量

2、初始化事件变量，设置其通知方式和通知信号

3、使用timer_create()函数创建定时器，设置定时器的属性和超时时间

4、在信号处理函数中处理定时器事件



*simple_threading_timer.c* 例子

在每一个超时中，会创建新线程，然后**expiration**函数被调用。这种方法的优点是代码占用空间小且调试简单，缺点是定时器超时时创建新线程而产生额外开销。



中断信号定时器

基于内核信号的定时器，内核在定时器每次超时不会创建一个新线程，而是向进程发送信号，进程被中断，并调用相应的信号处理程序。

基于内核信号的定时器实现起来有点复杂。



# 六、参考链接

**linux定时器编程详解（包含代码）**

https://zhuanlan.zhihu.com/p/374833099



https://www.lmlphp.com/user/59168/article/item/1363779/



linux定时器解析

https://opensource.com/article/21/10/linux-timers
