#### 生产者与消费者模式

-   [代码介绍](https://blog.csdn.net/marble_xu/article/details/98197962#_1)
-   -   [测试效果](https://blog.csdn.net/marble_xu/article/details/98197962#_17)
-   [完整测试代码](https://blog.csdn.net/marble_xu/article/details/98197962#_20)

## 代码介绍

学习[TDengine](https://so.csdn.net/so/search?q=TDengine&spm=1001.2101.3001.7020) [tsched.c](https://github.com/taosdata/TDengine/tree/master/src/util/src/tsched.c) 中的代码，这是多个生产者和消费者的模式，所以基于单个生产者和消费者的忙等待方式是不适合的，会消耗cpu资源，必须要用到信号量。

具体的实现方式：

-   使用两个信号量和一个互斥锁。
-   互斥锁 queueMutex 用来保护queue中的数据， 生产者和消费者在访问queue前都需要获取这个互斥锁。
-   信号量 emptySem 表示queue中是否有空的位置，可以放入新数据。
-   信号量 fullSem 表示queue中是否有数据，可以被读取。
-   每个消费者和生产者都是一个单独的线程。

生产者的处理循环图如下：  
![生产者](https://img-blog.csdnimg.cn/20190814095549748.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21hcmJsZV94dQ==,size_16,color_FFFFFF,t_70)  
消费者的处理循环图如下：  
![消费者](https://img-blog.csdnimg.cn/20190814100850680.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21hcmJsZV94dQ==,size_16,color_FFFFFF,t_70)  
对原来的代码进行了简单的修改，用来测试代码。

### 测试效果

为了能够看到正确的线程执行顺序，不能直接使用printf函数，所以使用一个全局的字符数组record，在每个线程读写数据的时候向这个全局数组添加内容。  
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190814100252411.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21hcmJsZV94dQ==,size_16,color_FFFFFF,t_70)

## 完整测试代码

在linux 下面运行测试。  
编译需要加上 -lpthread 选项。例子如下：  
gcc -o producer\_consumer producer\_consumer.c -lpthread

```
#include <errno.h>
#include <pthread.h>
#include <semaphore.h>
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <sys/syscall.h>

typedef struct
{
char data[16];
int num;
}SSchedMsg;

typedef struct 
{
sem_temptySem;
sem_tfullSem;
pthread_mutex_tqueueMutex;
intfullSlot;
intemptySlot;
intqueueSize;
intnumOfThreads;
SSchedMsg *queue;
pthread_t *qthread;
} SSchedQueue;

#define RECORD_LEN 1024
static charrecord[RECORD_LEN];
static intrecord_len = RECORD_LEN-1;
static char * ptr;
static int done = 0;

void initRecord()
{
memset(record, 0, RECORD_LEN);
ptr = (char *)record; 
}

/* type value 0:producer, 1:consumer */
void recordAction(int tid, int type, int num)
{
int len;

if(record_len <= 0)
{
done = 1;
return;
}

if(type == 0)
len = snprintf(ptr, record_len, "Producer[%d] set queue index[%d]\n",tid, num);
else
len = snprintf(ptr, record_len, "\t\tConsumer[%d] get queue index[%d]\n", tid, num);
record_len -= len;
ptr += len;
}

void * produceSchedQueue(void * param);
void * consumeSchedQueue(void *param);
void cleanupScheduler(void *param);

void * initScheduler(int queueSize, int numOfThreads)
{
int i = 0;
pthread_attr_t attr;
SSchedQueue * pSched = (SSchedQueue *)malloc(sizeof(SSchedQueue));

if(pSched == NULL) return NULL;

memset(pSched, 0, sizeof(SSchedQueue));
pSched->queueSize = queueSize;
pSched->numOfThreads = numOfThreads;

if(pthread_mutex_init(&pSched->queueMutex, NULL) < 0)
{
printf("init queueMutex failed, reason:%s", strerror(errno));
goto Error;
}

if(sem_init(&pSched->emptySem, 0, (unsigned int)pSched->queueSize) != 0)
{
printf("init empty semaphore failed, reason:%s", strerror(errno));
goto Error;
}

if(sem_init(&pSched->fullSem, 0, 0) != 0)
{
printf("init full semaphore failed, reason:%s", strerror(errno));
goto Error;
}

if((pSched->queue = (SSchedMsg *)malloc((size_t)pSched->queueSize * sizeof(SSchedMsg))) == NULL)
{
printf("no enough memory for queue, reason:%s", strerror(errno));
goto Error;
}
memset(pSched->queue, 0, (size_t)pSched->queueSize * sizeof(SSchedMsg));

pSched->fullSlot = 0;
pSched->emptySlot = 0;
pSched->qthread = malloc(sizeof(pthread_t) * (size_t)pSched->numOfThreads);

pthread_attr_init(&attr);
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);

for(i = 0; i < pSched->numOfThreads; i++)
{
void * (*func)(void *);
if(i % 2 == 0)
func = consumeSchedQueue;
else
func = produceSchedQueue;

if(pthread_create(pSched->qthread + i, &attr, func, (void *)pSched) != 0)
{
printf("failed to create rpc thread[%d], reason:%s", i, strerror(errno));
goto Error;
}
}
printf("scheduler is initialized, numOfThreads:%d\n", pSched->numOfThreads);
return (void *)pSched;

Error:
cleanupScheduler((void *)pSched);
return NULL;
}

pid_t gettid()
{
     return syscall(SYS_gettid);
}

void * consumeSchedQueue(void * param)
{
SSchedMsg msg;
SSchedQueue * pSched = (SSchedQueue *)param;
pid_t tid = gettid();

if(pSched == NULL)
{
printf("SSchedQueue is NULL");
return NULL;
}

while(1)
{
if(sem_wait(&pSched->fullSem) != 0)
{
printf("wait fullSem failed, errno:%d, reason:%s", errno, strerror(errno));
if(errno == EINTR)
continue;
}

if(pthread_mutex_lock(&pSched->queueMutex) != 0)
printf("lock queueMutex failed, reason:%s", strerror(errno));

msg = pSched->queue[pSched->fullSlot];
memset(pSched->queue + pSched->fullSlot, 0, sizeof(SSchedMsg));
pSched->fullSlot = (pSched->fullSlot + 1) % pSched->queueSize;

recordAction(tid, 1, msg.num);

if(pthread_mutex_unlock(&pSched->queueMutex) != 0)
printf("unlock queueMutex failed, reason:%s\n", strerror(errno));

if(sem_post(&pSched->emptySem) != 0)
printf("post emptySem failed, reason:%s\n", strerror(errno));

usleep(10000);
}
}

void * produceSchedQueue(void * param)
{
SSchedMsg msg;
SSchedQueue * pSched = (SSchedQueue *)param;
pid_t tid = gettid();

if(pSched == NULL)
{
printf("SSchedQueue is NULL");
return NULL;
}

memset(&msg, 0, sizeof(SSchedMsg));

while(1)
{
if(sem_wait(&pSched->emptySem) != 0)
{
printf("wait emptySem failed, errno:%d, reason:%s", errno, strerror(errno));
if(errno == EINTR)
continue;
}

if(pthread_mutex_lock(&pSched->queueMutex) != 0)
printf("lock queueMutex failed, reason:%s", strerror(errno));

msg.num = pSched->emptySlot;
pSched->queue[pSched->emptySlot] = msg;
pSched->emptySlot = (pSched->emptySlot + 1) % pSched->queueSize;

recordAction(tid, 0, msg.num);

if(pthread_mutex_unlock(&pSched->queueMutex) != 0)
printf("unlock queueMutex failed, reason:%s\n", strerror(errno));

if(sem_post(&pSched->fullSem) != 0)
printf("post fullSem failed, reason:%s\n", strerror(errno));

usleep(10000);
}
}

void cleanupScheduler(void * param)
{
int i = 0;
SSchedQueue * pSched = (SSchedQueue *)param;

if(pSched == NULL) return;

for( i = 0; i < pSched->numOfThreads; i++)
{
pthread_cancel(pSched->qthread[i]);
pthread_join(pSched->qthread[i], NULL);
}

sem_destroy(&pSched->emptySem);
sem_destroy(&pSched->fullSem);
pthread_mutex_destroy(&pSched->queueMutex);

free(pSched->qthread);
free(pSched->queue);
free(pSched);
}

int main()
{
initRecord();
initScheduler(6, 8);
while(1)
{
if(done)
{
printf("%s", record);
break;
}
}
}
```