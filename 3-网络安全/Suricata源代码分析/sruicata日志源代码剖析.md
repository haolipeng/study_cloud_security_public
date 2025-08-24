suricata默认的日志路径/var/log/suricata





日志类型

将模块注册到output_modules全局变量中

output_modules

```
OutputRegisterModule
调用者：
LuaLogRegister 我们暂时不关心lua方式自定义日志
OutputJsonRegister

OutputRegisterPacketModule
OutputRegisterPacketSubModule

OutputRegisterTxModuleWrapper
OutputRegisterTxSubModuleWrapper

OutputRegisterFileModule
OutputRegisterFileSubModule

OutputRegisterFiledataModule
OutputRegisterFiledataSubModule

OutputRegisterFlowSubModule

OutputRegisterStreamingModule
OutputRegisterStreamingSubModule

OutputRegisterStatsModule
OutputRegisterStatsSubModule
```



调用OutputRegisterFlowSubModule的函数

JsonFlowLogRegister

JsonNetFlowLogRegister



NetFlow和Flow的区别是什么？





**初始化output_modules**

RunModeInitializeOutputs

RunModeInitializeEveOutput



通过名称获取指定Module

OutputGetModuleByConfName



OutputRegisterTxModuleWrapper函数中，申请output模块，并注册到output_modules全局变量中。



在outputs节点上，可以设置如下的内容。

1、fast log

默认是开启的，日志文件名称为fast.log

用途：记录alert告警。



2、eve-log

默认是开启的，日志文件名称为eve.json



3、http-log

suricata配置文件中http-log选项如下：

```
- http-log:                     #The log-name.
    enabled: yes                #This log is enabled. Set 'no' to disable.
    filename: http.log          #The name of the file in the default logging directory.
    append: yes/no              #If this option is set to yes, the last filled http.log-file will not be
                                # overwritten while restarting Suricata.
    extended: yes               # If set to yes more information is written about the event.
```

当开启extended标志时，会记录更多事件信息。

未开启extended标志的http log：

```
07/01/2014-04:20:14.338309 vg.no [**] / [**] Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_2)
AppleWebKit/537.36 (KHTML, like Gecko) Chrome/35.0.1916.114 Safari/537.36 [**]
192.168.1.6:64685 -> 195.88.54.16:80
```



已开启extended标志的http log：

```
07/01/2014-04:21:06.994705 vg.no [**] / [**] Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_2)
AppleWebKit/537.36 (KHTML, like Gecko) Chrome/35.0.1916.114 Safari/537.36 [**] <no referer> [**]
GET [**] HTTP/1.1 [**] 301 => http://www.vg.no/ [**] 239 bytes [**] 192.168.1.6:64726 -> 195.88.54.16:80
```



4、tls-log



5、dns-log

```
dns-log:
  enabled: yes
  filename: dns.log
  append: yes
  types:
    - query
    - response
```



6、pcap-log

开启此选项，可以保存所有packet数据包。

suricata配置文件中pcap-log选项如下：

```
- pcap-log:
    enabled:  yes
    filename: log.pcap

    # Limit in MB.
    limit: 32 #限制pcap文件的大小

    mode: sguil # "normal" (default) or sguil.
    sguil_base_dir: /nsm_data/
```

默认情况下记录所有数据包，除了：

- 超过tcp流重组深度的tcp流
- 密钥交换后的加密流量



7、alert-debug

一种可提供关于警报的补充信息的日志类型。对于调查规则误报和编写规则的人特别方便，但是它会降低程序的性能。



8、alert-prelude



suricata.yaml中tls-store配置项

eve.json日志文件是如何创建的？



真正去输出flow log的函数为OutputFlowLog

InitFunc是何时调用

InitSubFunc何时调用



关键数据结构，注册的日志输出器

registered_loggers

```
void OutputRegisterRootLogger(ThreadInitFunc ThreadInit,
    ThreadDeinitFunc ThreadDeinit,
    ThreadExitPrintStatsFunc ThreadExitPrintStats,
    OutputLogFunc LogFunc, OutputGetActiveCountFunc ActiveCntFunc)
{
    RootLogger *logger = SCCalloc(1, sizeof(*logger));
    logger->ThreadInit = ThreadInit;
    logger->ThreadDeinit = ThreadDeinit;
    logger->ThreadExitPrintStats = ThreadExitPrintStats;
    logger->LogFunc = LogFunc;
    logger->ActiveCntFunc = ActiveCntFunc;
    //注册新logger到registered_loggers中
    TAILQ_INSERT_TAIL(&registered_loggers, logger, entries);
}
```

这个函数要好好弄清楚。



OutputLoggerLog也是很重要的函数，其调用的函数如下：

FlowWorkerStreamTCPUpdate

FlowWorkerFlowTimeout

FlowWorker