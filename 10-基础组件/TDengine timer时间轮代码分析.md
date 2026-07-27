# 一、内存池代码

结构体定义

```
// 内存池结构体
typedef struct {
	int				numOfFree; // 可用内存块数量
	int 			first; // 第一个可用内存块的索引
	int 			numOfBlock; // 内存块总数
	int 			blockSize; // 每个内存块的大小
	int *			freeList; // 可用内存块列表
	char * 			pool; // 内存池
	pthread_mutex_t	mutex; // 互斥锁，用于同步
} pool_t;
```



```
// 内存池初始化函数
mpool_h memPoolInit(int maxNum, int blockSize);

// 内存池分配函数
char * memPoolMalloc(mpool_h handle);

// 内存池释放函数
void memPoolFree(mpool_h handle, char *p);

// 内存池清理函数
void memPoolCleanup(mpool_h handle);

mpool_h memPoolInit(int numOfBlock, int blockSize)
{
	int i = 0;
	pool_t * pool_p = NULL;

	if(numOfBlock <= 1 || blockSize <= 1)
	{
		printf("invalid parameter in memPoolInit\n");
		return NULL;
	}

	pool_p = (pool_t *)malloc(sizeof(pool_t));
	if(pool_p == NULL)
	{
		printf("mempool malloc failed\n");
		return NULL;
	}

	memset(pool_p, 0, sizeof(pool_t));

	pool_p->blockSize = blockSize;
	pool_p->numOfBlock = numOfBlock;
	pool_p->pool = (char *)malloc((size_t)(blockSize * numOfBlock));
	pool_p->freeList = (int *)malloc(sizeof(int) * (size_t)numOfBlock);

	if(pool_p->pool == NULL || pool_p->freeList == NULL)
	{
		printf("failed to allocate memory\n");
		free(pool_p->freeList);
		free(pool_p->pool);
		free(pool_p);
	}

	pthread_mutex_init(&(pool_p->mutex), NULL);

	for(i = 0; i < pool_p->numOfBlock; i++)
		pool_p->freeList[i] = i;

	pool_p->first = 0;
	pool_p->numOfFree= pool_p->numOfBlock;

	return (mpool_h)pool_p;
}

char * memPoolMalloc(mpool_h handle)
{
	char * pos = NULL;
	pool_t * pool_p = (pool_t *)handle;

	pthread_mutex_lock(&pool_p->mutex);

	if(pool_p->numOfFree <= 0)
	{
		printf("mempool: out of memory");
	}
	else
	{
		pos = pool_p->pool + pool_p->blockSize * (pool_p->freeList[pool_p->first]);
		pool_p->first = (pool_p->first + 1) % pool_p->numOfBlock;
		pool_p->numOfFree--;
	}

	pthread_mutex_unlock(&pool_p->mutex);
	if(pos != NULL) memset(pos, 0, (size_t)pool_p->blockSize);
	return pos;
}

void memPoolFree(mpool_h handle, char * pMem)
{
	int index = 0;
	pool_t * pool_p = (pool_t *)handle;

	if(pool_p == NULL || pMem == NULL) return;

	pthread_mutex_lock(&pool_p->mutex);

	index = (int)(pMem - pool_p->pool) % pool_p->blockSize;
	if(index != 0)
	{
		printf("invalid free address:%p\n", pMem);
	}
	else
	{
		index = (int)((pMem - pool_p->pool) / pool_p->blockSize);
		if(index < 0 || index >= pool_p->numOfBlock)
		{
			printf("mempool: error, invalid address:%p\n", pMem);
		}
		else
		{
			pool_p->freeList[(pool_p->first + pool_p->numOfFree) % pool_p->numOfBlock] = index;
			pool_p->numOfFree++;
			memset(pMem, 0, (size_t)pool_p->blockSize);
		}
	}
	
	pthread_mutex_unlock(&pool_p->mutex);
}

void memPoolCleanup(mpool_h handle)
{
	pool_t *pool_p = (pool_t *)handle;

	pthread_mutex_destroy(&pool_p->mutex);
	if(pool_p->pool) free(pool_p->pool);
	if(pool_p->freeList) free(pool_p->freeList);
}
```



# 二、时间轮定时器

## 2、1 定义结构体

定时器对象结构体

定时器链表结构体

定时器控制器结构体



```
#include <errno.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <sys/time.h>


#define mpool_h void *
typedef void *tmr_h;

#define maxNumOfTmrCtrl  	16
#define MSECONDS_PER_TICK 	5

// 定时器对象结构体
typedef struct _tmr_obj
{
	void *param1; // 参数1
	void (*fp)(void *, void *); // 回调函数
	tmr_h				timerId; // 定时器ID
	short				cycle; // 周期
	struct _tmr_obj * 	prev; // 前一个定时器对象
	struct _tmr_obj * 	next; // 后一个定时器对象
	int                 index; // 索引,通过索引来找到定时器
	struct _tmr_ctrl_t *pCtrl; // 定时器控制器
} tmr_obj_t;

// 定时器链表结构体
typedef struct
{
	tmr_obj_t * head; // 定时器对象链表头
	int			count; // 定时器对象数量
} tmr_list_t;

// 定时器控制器结构体
typedef struct _tmr_ctrl_t
{
	void * 			signature; // 签名
	pthread_mutex_t mutex; // 互斥锁，用于同步
	int				resolution; // 分辨率
	int 			numOfPeriods; // 周期数
	unsigned int	periodsFromStart; // 从开始到现在的周期数
	tmr_list_t *	tmrList; // 定时器列表
	mpool_h			poolHandle; // 内存池句柄
	char            label[12]; // 标签，表明是什么定时器
	int 			maxNumOfTmrs; // 最大定时器数量
	int 			numOfTmrs; // 当前定时器数量
	int 			ticks; // tick数
	int				maxTicks; // 最大tick数
} tmr_ctrl_t;

//存储定时器控制器的数组
tmr_ctrl_t tmrCtrl[maxNumOfTmrCtrl];

// 处理定时信号的函数
void * processAlarmSignal(void *tharg)
{
	sigset_t 		sigset;
	timer_t	 		timerId;
	int signo;

	sigemptyset(&sigset);
	sigaddset(&sigset, SIGALRM);
	sigprocmask(SIG_BLOCK, &sigset, NULL);

	struct itimerval new_value, old_value;
	new_value.it_value.tv_sec = 0;
	new_value.it_value.tv_usec = 1000 * MSECONDS_PER_TICK;
	new_value.it_interval.tv_sec = 0;
	new_value.it_interval.tv_usec = 1000 * MSECONDS_PER_TICK;
	setitimer(ITIMER_REAL, &new_value, &old_value);

	//循环等待处理alarm信号
	while(1)
	{
		//TODO:这里会阻塞吗?
		if(sigwait(&sigset, &signo))
		{
	      	printf("Failed to wait signal: number %d", signo);
	      	continue;
	    }
		timerLoopFunc();
	}
	return NULL;
}

// 定时器模块初始化函数
void tmrModuleInit(void)
{
	pthread_t		thread;
	pthread_attr_t	tattr;
	
	memset(tmrCtrl, 0, sizeof(tmrCtrl));

	pthread_attr_init(&tattr);
	pthread_attr_setdetachstate(&tattr, PTHREAD_CREATE_DETACHED);//设置线程状态
	if(pthread_create(&thread, &tattr, processAlarmSignal, NULL) != 0)
	{
		printf("failed to create timer thread");
    	return;
  	}

  	pthread_attr_destroy(&tattr);
}
```



## 2、2 定时器初始化

tmrInit函数是用来初始化定时器控制器的。它接收四个参数：最大定时器数量、分辨率、最长周期和标签。

```
// 定时器初始化函数
void * tmrInit(int maxNumOfTmrs, int resolution, int longest, char * label)
{
	tmr_ctrl_t * pCtrl; // 定义定时器控制器指针
	int tmrCtrlId = 0; // 定义定时器控制器ID
	int i = 0; // 定义循环变量

	pCtrl = tmrCtrl + tmrCtrlId; // 获取定时器控制器

	memset(pCtrl, 0, sizeof(tmr_ctrl_t)); // 清零定时器控制器内存
	strncpy(pCtrl->label, label, sizeof(pCtrl->label)); // 设置标签
	pCtrl->maxNumOfTmrs = maxNumOfTmrs; // 设置最大定时器数量

	// 初始化内存池，如果失败则清理内存池并返回NULL
	if((pCtrl->poolHandle = memPoolInit(maxNumOfTmrs + 10,sizeof(tmr_obj_t))) == NULL)
	{
		printf("%s failed to allocate mem pool", label);
		memPoolCleanup(pCtrl->poolHandle);
		return NULL;
	}
	
	// 设置定时器控制器的分辨率，最大tick数，随机的tick数，周期数
	if(resolution < MSECONDS_PER_TICK) resolution = MSECONDS_PER_TICK;
	pCtrl->resolution = resolution;
	pCtrl->maxTicks = resolution / MSECONDS_PER_TICK;
	pCtrl->ticks = rand() / pCtrl->maxTicks;
	pCtrl->numOfPeriods = longest / resolution;
	if(pCtrl->numOfPeriods < 10) pCtrl->numOfPeriods = 10;

	// 为定时器列表分配内存，并初始化每个定时器列表的头和数量
	pCtrl->tmrList = (tmr_list_t *)malloc(sizeof(tmr_list_t) * pCtrl->numOfPeriods);
	for(i = 0; i < pCtrl->numOfPeriods; i++)
	{
		pCtrl->tmrList[i].head = NULL;
		pCtrl->tmrList[i].count = 0;
	}

	pCtrl->signature = pCtrl; // 设置定时器控制器的签名
	printf("%s timer ctrl is initialized, tmrCtrlId:%d\n", label, tmrCtrlId); // 打印初始化信息
	return pCtrl; // 返回定时器控制器的指针
}
```

在函数中，首先清零定时器控制器的内存，然后设置标签和最大定时器数量。接着，初始化内存池，如果失败则清理内存池并返回NULL。

然后，设置定时器控制器的分辨率，最大tick数，随机的tick数，周期数。如果周期数小于10，则设置为10。

接着，为定时器列表分配内存，并初始化每个定时器列表的头和数量。

最后，设置定时器控制器的签名，并打印初始化信息，返回定时器控制器的指针。



## 2、3 重置定时器函数

tmrReset函数是用来重置定时器的。它接收五个参数：回调函数、毫秒数、参数1、句柄和定时器ID指针。

```
// 重置定时器函数
void tmrReset(void (*fp)(void *, void*), int mseconds, void * param1, void * handle, tmr_h *pTmrId)
{
	tmr_obj_t *pObj, *cNode, *pNode;
	tmr_list_t * pList = NULL;
	int index, period;
	tmr_ctrl_t *pCtrl = (tmr_ctrl_t *)handle;

	if(handle == NULL || pTmrId == NULL) return; // 如果句柄或定时器ID为空，直接返回

	period = mseconds / pCtrl->resolution; // 计算周期
	if(pthread_mutex_lock(&pCtrl->mutex) != 0) // 加锁，防止其他线程同时操作
		printf("%s mutex lock failed, reason:%s", pCtrl->label, strerror(errno));

	pObj = (tmr_obj_t *)(*pTmrId); // 获取定时器对象

	// 如果定时器对象存在且其ID与传入的定时器ID相同
	if(pObj && pObj->timerId == *pTmrId)
	{
		pList = &(pCtrl->tmrList[pObj->index]); // 获取定时器列表
		if(pObj->prev)
			pObj->prev->next = pObj->next; // 如果定时器对象的前一个对象存在，将其下一个对象设置为当前对象的下一个对象
		else
			pList->head = pObj->next; // 否则，将定时器列表的头设置为当前对象的下一个对象

		if(pObj->next)
			pObj->next->prev = pObj->prev; // 如果定时器对象的下一个对象存在，将其前一个对象设置为当前对象的前一个对象

		pList->count--; // 定时器列表数量减1
		pObj->timerId = NULL; // 清空定时器对象的ID
		pCtrl->numOfTmrs--; // 定时器数量减1
		printf("reset pObj:%p\n", pObj); // 打印重置信息
	}
	else // 如果定时器对象不存在或其ID与传入的定时器ID不同
	{
		pObj = (tmr_obj_t *)memPoolMalloc(pCtrl->poolHandle); // 从内存池中分配内存
		*pTmrId = pObj; // 设置定时器ID
		if(pObj == NULL) // 如果分配失败
		{
			printf("%s failed to allocate timer, max:%d allocated:%d", pCtrl->label, pCtrl->maxNumOfTmrs, pCtrl->numOfTmrs);
      		pthread_mutex_unlock(&pCtrl->mutex); // 解锁
      		return; // 返回
		}
		printf("malloc pObj:%p\n", pObj); // 打印分配信息
	}

	// 设置定时器对象的周期、参数1、回调函数、ID、控制器
	pObj->cycle = period / pCtrl->numOfPeriods;
	pObj->param1 = param1;
	pObj->fp = fp;
	pObj->timerId = pObj;
	pObj->pCtrl = pCtrl;

	// 计算索引
	index = (period + pCtrl->periodsFromStart) % pCtrl->numOfPeriods;
	pList = &(pCtrl->tmrList[index]);
	pObj->index = index;
	cNode = pList->head;
	pNode = NULL;

	// 将定时器对象插入到定时器列表中的正确位置
	while(cNode != NULL)
	{
		if(cNode->cycle < pObj->cycle)
		{
			pNode = cNode;
			cNode = cNode->next;
		}
		else
			break;
	}

	pObj->next = cNode;
	pObj->prev = pNode;

	if(cNode != NULL)
		cNode->prev = pObj;

	if(pNode != NULL)
		pNode->next = pObj;
	else
		pList->head = pObj;

	pList->count++;
	pCtrl->numOfTmrs++;

	if (pthread_mutex_unlock(&pCtrl->mutex) != 0) // 解锁
    	printf("%s mutex unlock failed, reason:%s", pCtrl->label, strerror(errno));

	printf("%s %p, timer is reset, fp:%p, tmr_h:%p, cycle:%d, index:%d, total:%d numOfFree:%d\n", pCtrl->label, param1, fp, pObj,
           pObj->cycle, index, pCtrl->numOfTmrs, ((pool_t *)pCtrl->poolHandle)->numOfFree);
	return;
}
```



## 2、4 定时器处理函数

这个函数的主要作用是处理定时器列表，遍历列表中的每一个定时器对象，分以下两种情况：

1、如果定时器的周期大于0，则周期减1，

2、如果定时器的周期等于0，表示定时器到期，此时会调用定时器的回调函数，并从定时器列表中移除该定时器对象。

```
// 定时器处理函数
void tmrProcessList(tmr_ctrl_t *pCtrl)
{
	int index; // 定义索引
	tmr_list_t * pList; // 定义定时器列表指针
	tmr_obj_t * pObj, *header; // 定义定时器对象指针

	pthread_mutex_lock(&pCtrl->mutex); // 加锁，防止其他线程同时操作
	index = pCtrl->periodsFromStart % pCtrl->numOfPeriods; // 计算索引
	pList = &pCtrl->tmrList[index]; // 获取定时器列表
	while(1)
	{
		header = pList->head; // 获取定时器列表头
		if(header == NULL) break; // 如果头为空，跳出循环

		if(header->cycle > 0) // 如果定时器周期大于0
		{
			pObj = header; // 获取定时器对象
			while(pObj) // 遍历定时器对象
			{
				pObj->cycle--; // 周期减1
				pObj = pObj->next; // 获取下一个定时器对象
			}
			break; // 跳出循环
		}

		pCtrl->numOfTmrs--; // 定时器数量减1
		printf("%s %p, timer expired, fp:%p, tmr_h:%p, index:%d, total:%d\n", pCtrl->label, header->param1, header->fp,
					 header, index, pCtrl->numOfTmrs); // 打印定时器过期信息

		pList->head = header->next; // 更新定时器列表头
		if(header->next) header->next->prev = NULL; // 如果头的下一个节点存在，将其前一个节点设置为NULL
		pList->count--; // 定时器列表数量减1
		header->timerId = NULL; // 清空定时器ID

		if (header->fp) // 如果回调函数存在
      		(*(header->fp))(header->param1, header); // 调用回调函数
		
		memPoolFree(pCtrl->poolHandle, (char *)header); // 释放内存池中的内存
	}
	
	pCtrl->periodsFromStart++; // 周期数增1
	pthread_mutex_unlock(&pCtrl->mutex); // 解锁
}

// 定时器循环函数
void * timerLoopFunc(void)
{
	tmr_ctrl_t *pCtrl;
	int i = 0;

	for(i = 0; i < maxNumOfTmrCtrl; i++)
	{
		pCtrl = tmrCtrl + i;
		if(pCtrl->signature)
		{
			pCtrl->ticks++;
			if(pCtrl->ticks >= pCtrl->maxTicks)
			{
				//调用定时器处理函数
				tmrProcessList(pCtrl);
				pCtrl->ticks = 0;
			}
		}
	}
}
```



# 三、测试代码

```
void timerFunc1(void *param, void *tmrId) 
{
	int id = *(int *)param;
	printf("%s id[%d]\n", __func__, id);
}

void timerFunc2(void *param, void *tmrId) 
{
	int id = *(int *)param;
	printf("%s id[%d]\n", __func__, id);
}

void test()
{	
	void *timerHandle = tmrInit(5, 100, 1000, "http");
	void *timer1 = NULL, *timer2 = NULL, *timer3 = NULL;
	int id1 = 1, id2 = 2, id3 = 3;

	tmrReset(timerFunc1, 100, (void *)&id1, timerHandle, &timer1);

	tmrReset(timerFunc2, 1100, (void *)&id2, timerHandle, &timer2);

	tmrReset(timerFunc2, 1800, (void *)&id3, timerHandle, &timer3);
}

int main()
{
	//这里这句话是在干什么呢?
	sigset_t 		sigset;

	sigemptyset(&sigset);
	sigaddset(&sigset, SIGALRM);
	sigprocmask(SIG_BLOCK, &sigset, NULL);
	
	tmrModuleInit();
	test();
	while(1)
	{	
	}
}
```

