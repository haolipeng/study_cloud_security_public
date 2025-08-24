技术博客-编写大纲

主要是解决先干什么，后干什么的问题。



1、搭建可单步debug的环境

2、suricata的收包模块

​	重点解析下三种模式，pcap、af_packet、dpdk，其他的模式带一笔即可。

3、基础网络协议解析

4、应用层协议识别

5、应用层协议解析，支持哪些协议 比如http协议，dns协议

6、suricata如何添加一个新的协议呢，比如我如何添加车联网协议CAN呢

7、



使用suricata

剖析suricata的实现方式

阅读suricata的源代码

移植源代码应用到其他项目中



编译

./configure --enable-unittests --enable-debug
