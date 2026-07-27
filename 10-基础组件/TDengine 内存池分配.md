#### 内存池分配

-   [代码介绍](https://blog.csdn.net/marble_xu/article/details/99544895#_1)
-   -   [内存池分配释放示例](https://blog.csdn.net/marble_xu/article/details/99544895#_23)
    -   [测试效果](https://blog.csdn.net/marble_xu/article/details/99544895#_37)
-   [完整测试代码](https://blog.csdn.net/marble_xu/article/details/99544895#_40)

## 代码介绍

学习[TDengine](https://so.csdn.net/so/search?q=TDengine&spm=1001.2101.3001.7020) [tmempool.c](https://github.com/taosdata/TDengine/tree/master/src/util/src/tmempool.c) 中的代码，主要学习下内存池的数据结构设计。  
这个有点类似于linux 内核中的kmem\_cache, 预先创建内存块大小为 blockSize，数量为numOfBlock的内存池，然后每次从内存池中获取大小为blockSize的内存块。

```
typedef struct {
  int             numOfFree;  /* number of free slots */
  int             first;      /* the first free slot  */
  int             numOfBlock; /* the number of blocks */
  int             blockSize;  /* block size in bytes  */
  int *           freeList;   /* the index list       */
  char *          pool;       /* the actual mem block */
  pthread_mutex_t mutex;
} pool_t;
```

结构体pool\_t中的成员

-   pool：是申请的连续内存空间，大小为 (blockSize \* numOfBlock)
-   freeList: 一个int 数组，每个项保存指向 pool 中的内存块的偏移index
-   first：第一个可用内存块的freeList数组index
-   numOfFree: 内存池中可用内存块的数目

freeList 可以看成是一个用来模拟链表的数组。index 在 \[first, first + numOfFree) 范围内的freeList 数组的值表示pool中可分配内存块的偏移index。这里的 first + numOfFree 值在作为freeList 数组index时实际是取除以numOfBlock 的余数，即 (first + numOfFree) % numOfBlock 的值。

### 内存池分配释放示例

假设创建了一个numOfBlock 为6 的内存池，通过一个例子来看下内存池的结构变化。

1.  初始化时的内存池如图1， first 值为0，numOfFree值为6，图中的 first + numOfFree 值实际是 (first + numOfFree) % numOfBlock，所以值为0。

图1![mempool1](https://img-blog.csdnimg.cn/20190815073607174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21hcmJsZV94dQ==,size_16,color_FFFFFF,t_70)

2.  用户从内存池中分配了3个内存块给a1，a2，a3以后的内存池如图2，first 目前值为3，numOfFree值为3，freeList 数组中红色字体的值表示已经分配出去的内存块偏移值。可以看到，分配内存时， (first + numOfFree) % numOfBlock 的值是保持不变的。

图2![mempool2](https://img-blog.csdnimg.cn/2019081507374632.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21hcmJsZV94dQ==,size_16,color_FFFFFF,t_70)

3.  用户按顺序释放了a3，a2 以后的内存池如图3，因为先释放了a3，所以freeList\[0\]的值设为2。释放内存时，first的值保持不变。

图3 ![在这里插入图片描述](https://img-blog.csdnimg.cn/2019081507484341.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21hcmJsZV94dQ==,size_16,color_FFFFFF,t_70)

### 测试效果

按照上面例子写的测试程序运行结果如下，和图示是一致的。  
![result](https://img-blog.csdnimg.cn/20190815093736845.png)

## 完整测试代码

在linux 下面运行测试。  
编译需要加上 -lpthread 选项。例子如下：  
gcc -o mempool mempool.c -lpthread

```
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define mpool_h void *

typedef struct {
   intnumOfFree;
   int first;
   int numOfBlock;
   int blockSize;
   int *freeList;
   char * pool;
   pthread_mutex_tmutex;
} pool_t;

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

void displayMemPool(mpool_h handle)
{
   int i = 0, index = 0;
   pool_t * pool_p = (pool_t *)handle;

   if(pool_p == NULL) return;

   printf("Mempool Info\nfirst:%d, numOfFree:%d\n", pool_p->first, pool_p->numOfFree);

   for(i = pool_p->first; i < (pool_p->first + pool_p->numOfBlock); i++)
   {
   index = i % pool_p->numOfBlock;
   if(i >= pool_p->first + pool_p->numOfFree)
   printf("\tfreeList[%d] => used\n", index);
   else
   printf("\tfreeList[%d] => pool[%d]\n", index, pool_p->freeList[index]);
   }
}

#define BLOCK_NUM 6
void testMemPool(mpool_h handle)
{
   int i = 0;

   displayMemPool(handle);

   void * a[BLOCK_NUM] = {0};
   for(i = 0; i < 3; i++)
   a[i] = memPoolMalloc(handle);

   displayMemPool(handle);
   
   memPoolFree(handle, (char *)a[2]);
   memPoolFree(handle, (char *)a[1]);

   displayMemPool(handle);
}

int main()
{
   void * pool = memPoolInit(BLOCK_NUM, 100);
   testMemPool(pool);
   memPoolCleanup(pool);
}
```