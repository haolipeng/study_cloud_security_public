现在未搞清楚的一些事情。

suricata的配置文件字段说明

https://www.cnblogs.com/UnGeek/p/5796934.html



采用cgdb的方式来调试suricata源代码。



协议插件架构是如何的？

协议识别是如何的？



参考文档：https://docs.suricata.io/en/latest/devguide/codebase/installation-from-git.html

# 零、预安装要求

在ubuntu环境下，其依赖库如下：

```
apt-get -y install libpcre2-dev build-essential autoconf \
automake libtool libpcap-dev libnet1-dev libyaml-0-2 libyaml-dev \
pkg-config zlib1g zlib1g-dev libcap-ng-dev libcap-ng0 make \
libmagic-dev libjansson-dev rustc cargo jq git-core
```



# 一、编译安装

## 1、1 configure命令

通过bundle.sh获取的suricata-update可执行程序

### 1、1、1 开启Debug调试和单元测试

执行如下的configure命令

```shell
CFLAGS="-g -O0" ./configure --prefix=/usr  --sysconfdir=/etc --localstatedir=/var  --enable-debug --enable-unittests
```



**--enable-debug :**开启debug模式

--enable-unittests：开启单元测试

**--prefix=/usr :**将Suricata二进制文件安装到/usr/bin/

**--sysconfdir=/etc :**将Suricata配置文件安装到/etc/suricata/

**--localstatedir=/var :**设置Suricata写日志到/var/log/suricata/





**configure命令**

将libhtp安装在suricata源代码目录**（推荐）**

./configure --prefix=/usr  --sysconfdir=/etc --localstatedir=/var  --enable-debug --enable-unittests



如果libhtp是自定义安装的，需要加上PKG_CONFIG_PATH

PKG_CONFIG_PATH=/usr/lib/pkgconfig



```shell
CFLAGS="-g -O0" ./configure --prefix=/usr  --sysconfdir=/etc --localstatedir=/var  --enable-debug --enable-unittests --enable-non-bundled-htp --with-libhtp-includes=/usr/include --with-libhtp-libraries=/usr/lib
```

在执行configure命令之前需要配置下PKG_CONFIG_PATH环境变量，让suricata能否寻找到libhtp的位置

设置CFLAGS标志是因为要编译可单步debug版本的代码。



开启DPDK组件的编译选项

```
CFLAGS="-g -O0" PKG_CONFIG_PATH="/home/work/dpdk-stable-24.11.1/dpdklib/lib/x86_64-linux-gnu/pkgconfig/" LT_SYS_LIBRARY_PATH="/home/work/dpdk-stable-24.11.1/dpdklib/lib/x86_64-linux-gnu" ./configure --prefix=/usr  --sysconfdir=/etc --localstatedir=/var --enable-dpdk --enable-debug --enable-unittests 
```



## 1、2 编译和安装 make && make install

#make install-rules :安装suricata提供的规则文件到/etc/suricata/rules目录下



make install-conf不仅执行常规的"make install"，还有自动创建必要的目录，以及suricata.yaml文件给你。

```
root@r630-PowerEdge-R630:/home/work/openSource/suricata-latest# make install-conf
install -d "/etc/suricata/"
install -d "/var/log/suricata/files"
install -d "/var/log/suricata/certs"
install -d "/var/run/"
install -m 770 -d "/var/run/suricata"
install -m 770 -d "/var/lib/suricata/data"
install -m 770 -d "/var/lib/suricata/cache/sgh"
```



## 1、3 安装rules规则

默认情况下规则是安装在`/usr/share/suricata/`目录的。

在此目录下，有两个文件classification.config和reference.config以及一个文件夹rules

疑问点：classification.config是什么作用？

疑问点：reference.config是什么作用？

疑问点：http-events.rules中分析一条具体规则

```
alert http any any -> any any (msg:"SURICATA HTTP request too many headers"; flow:established,to_server; app-layer-event:http.request_too_many_headers; classtype:protocol-command-decode; sid:2221056; rev:1;)
alert http any any -> any any (msg:"SURICATA HTTP response too many headers"; flow:established,to_client; app-layer-event:http.response_too_many_headers; classtype:protocol-command-decode; sid:2221057; rev:1;)
```

请求中有太多header，响应中有太多header。



# 二、运行程序

```
oroot@r630-PowerEdge-R630:/home/work/openSource/suricata-latest# suricata --help
Suricata 8.0.1-dev (b93a27722 2025-08-07)
USAGE: suricata [OPTIONS] [BPF FILTER]


  General:
	-v                                   : be more verbose (use multiple times to increase verbosity)
	-c <path>                            : path to configuration file
	-l <dir>                             : default log directory
	--include <path>                     : additional configuration file
	--set name=value                     : set a configuration value
	--pidfile <file>                     : write pid to this file
	-T                                   : test configuration file (use with -c)
	--init-errors-fatal                  : enable fatal failure on signature init error
	-D                                   : run as daemon
	--user <user>                        : run suricata as this user after init
	--group <group>                      : run suricata as this group after init
	--unix-socket[=<file>]               : use unix socket to control suricata work
	--runmode <runmode_id>               : specific runmode modification the engine should run.  The argument
	                                       supplied should be the id for the runmode obtained by running
	                                       --list-runmodes

  Capture and IPS:
	-F <bpf filter file>                 : bpf filter file
	-k [all|none]                        : force checksum check (all) or disabled it (none)
	-i <dev or ip>                       : run in pcap live mode
	--pcap[=<dev>]                       : run in pcap mode, no value select interfaces from suricata.yaml
	--pcap-buffer-size                   : size of the pcap buffer value from 0 - 2147483647
	--af-packet[=<dev>]                  : run in af-packet mode, no value select interfaces from suricata.yaml
	--af-xdp[=<dev>]                     : run in af-xdp mode, no value select interfaces from suricata.yaml
	--reject-dev <dev>                   : send reject packets from this interface

  Capture Files:
	-r <path>                            : run in pcap file/offline mode
	--pcap-file-continuous               : when running in pcap mode with a directory, continue checking directory for pcaps until interrupted
	--pcap-file-delete                   : when running in replay mode (-r with directory or file), will delete pcap files that have been processed when done
	--pcap-file-recursive                : will descend into subdirectories when running in replay mode (-r)
	--pcap-file-buffer-size              : set read buffer size (setvbuf)
	--erf-in <path>                      : process an ERF file

  Detection:
	-s <path>                            : path to signature file loaded in addition to suricata.yaml settings (optional)
	-S <path>                            : path to signature file loaded exclusively (optional)
	--disable-detection                  : disable detection engine
	--engine-analysis                    : print reports on analysis of different sections in the engine and exit.
	                                       Please have a look at the conf parameter engine-analysis on what reports
	                                       can be printed

  Firewall:
	--firewall                           : enable firewall mode
	--firewall-rules-exclusive=<path>    : path to firewall rule file loaded exclusively

  Info:
	-V                                   : display Suricata version
	--list-keywords[=all|csv|<kword>]    : list keywords implemented by the engine
	--list-runmodes                      : list supported runmodes
	--list-app-layer-protos              : list supported app layer protocols
	--list-app-layer-hooks               : list supported app layer hooks for use in rules
	--dump-config                        : show the running configuration
	--dump-features                      : display provided features
	--build-info                         : display build information

  Testing:
	--simulate-ips                       : force engine into IPS mode. Useful for QA
	-u                                   : run the unittests and exit
	-U=REGEX, --unittest-filter=REGEX    : filter unittests with a pcre compatible regex
	--list-unittests                     : list unit tests
	--fatal-unittests                    : enable fatal failure on unittest error
	--unittests-coverage                 : display unittest coverage report


To run Suricata with default configuration on interface eth0 with signature file "signatures.rules", run the command as:

suricata -c suricata.yaml -s signatures.rules -i eth0
```

## 2、1 运行主程序

**生产环境**

```
suricata -c suricata.yaml
```

把所有配置都维护在yaml文件中，更容易管理和版本控制，符合最佳实践。



**测试环境**

```
suricata -c suricata.yaml -s signatures.rules -i eth0
suricata -c suricata.yaml -i eth0
```

- 方便快速测试新规则
- 方便切换监听接口
- 不影响生产配置



## 2、2 运行测试程序

内置的单元测试程序只有在configure时添加了--enable-unittests标志才有效。

运行单元测试程序不要求配置文件。



**单独执行某一个单元测试**

```
suricata -u -U StreamTcpTest14
```

运行的单元测试的名称为StreamTcpTest14，此时可以在StreamTcpTest14函数中下断点进行debug单步调试。



**-u：**运行单元测试程序然后退出。

**-U，--unittest-filter=REGEX**  使用-U选项，您可以选择要运行哪个单元测试。此选项使用正则表达式进行选择。

**--list-unittests**：列举出可用的单元测试

**--fatal-unittests**：当单元测试失败时，将会产生错误。



然后从命令行设置debug级别

```
SC_LOG_LEVEL=Debug suricata -u
```





# 三、容器化方式部署suricata

以容器化的方式来启动suricata程序。

```
docker run --rm -it --net=host --cap-add=net_admin --cap-add=net_raw --cap-add=sys_nice -v $(pwd)/logs:/var/log/suricata jasonish/suricata:latest -i ens34
```

