关于获取数据包的时间

DPDK 提供了更高效的时间获取API，主要使用 TSC (Time Stamp Counter) 寄存器：



timespec time;

clock_gettime(CLOCK_REALTIME, &time);



实现这个功能，大致可分为几步呢？

每一步是什么含义？