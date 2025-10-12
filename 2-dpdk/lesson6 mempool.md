# 一、为什么需要内存池

数据包在接收以后需要申请定长的内存空间

维护tcp会话表或者ip会话表的时候，会话表的元素，也是需要内存空间的。

频繁的申请和释放内存空间是会导致内存碎片以及处理性能的下降的。



# 二、内存池的基本操作

## 2、1 创建内存池

rte_mempool_create

函数原型如下：

```
struct rte_mempool * rte_mempool_create	(	const char * 	name,
    unsigned 	n,
    unsigned 	elt_size,
    unsigned 	cache_size,
    unsigned 	private_data_size,
    rte_mempool_ctor_t * 	mp_init,
    void * 	mp_init_arg,
    rte_mempool_obj_cb_t * 	obj_init,
    void * 	obj_init_arg,
    int 	socket_id,
    unsigned 	flags 
)	
```

这个函数需要重点讲解下，其他函数都比较简单，随意就行。

flags:标志位，

RTE_MEMPOOL_F_NO_SPREAD：

// 默认行为：对象地址在内存通道间分散（默认即可）

// 设置此标志：对象地址不分散，只按缓存行对齐



RTE_MEMPOOL_F_NO_CACHE_ALIGN

// 默认行为：对象按缓存行对齐

// 设置此标志：对象不按缓存行对齐，无填充

- 设置后：对象不按缓存行对齐，无填充，节省内存



RTE_MEMPOOL_F_SP_PUT

// 默认：多生产者模式

// 设置后：单生产者模式

适用场景：

- 只有一个线程向内存池归还对象时

- 需要最高性能的 put 操作时



RTE_MEMPOOL_F_SC_GET

// 默认：多消费者模式

// 设置后：单消费者模式

适用场景：

- 只有一个线程从内存池获取对象时

- 需要最高性能的 get 操作时



在rte_mempool_create_empty函数中

```
struct rte_mempool *
rte_mempool_create_empty(const char *name, unsigned n, unsigned elt_size,unsigned cache_size, unsigned private_data_size,
	int socket_id, unsigned flags)
{
	//省略N多代码
	if ((flags & RTE_MEMPOOL_F_SP_PUT) && (flags & RTE_MEMPOOL_F_SC_GET))
		ret = rte_mempool_set_ops_byname(mp, "ring_sp_sc", NULL);
	else if (flags & RTE_MEMPOOL_F_SP_PUT)
		ret = rte_mempool_set_ops_byname(mp, "ring_sp_mc", NULL);
	else if (flags & RTE_MEMPOOL_F_SC_GET)
		ret = rte_mempool_set_ops_byname(mp, "ring_mp_sc", NULL);
	else
		ret = rte_mempool_set_ops_byname(mp, "ring_mp_mc", NULL);
	//省略N多代码
}
```



备注：

rte_mempool_create和rte_mempool_create_empty函数的区别是什么呢？



## 2、2  从内存池获取元素get

在官网文档中有三个api：

rte_mempool_generic_get

rte_mempool_get_bulk 

rte_mempool_get





## 2、3 将元素归还给内存池put

rte_mempool_generic_put

rte_mempool_put_bulk

rte_mempool_put



## 2、4 统计元素使用信息counter

内存池可用元素

rte_mempool_avail_count



内存池已使用的元素

rte_mempool_in_use_count



## 2、5 释放内存池

rte_mempool_free



# 三、技术疑问点

疑问：有cache的mempool和没有cache的mempool有什么区别呢？