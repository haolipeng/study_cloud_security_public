为啥要替换mac地址呢？

# 一、基础信息收集

首先，找到容器对应哪个vin和vex接口

nginx容器：其pid为23503，对应的enforcer容器中vin和vex接口为

```
vex5bcf-eth0
vin5bcf-eth0
```

**ip地址：**172.17.0.3

**mac地址：**02:42:ac:11:00:03



centos容器：其pid为25876，对应的enforcer容器中vin和vex接口为

```
vex6514-eth0
vin6514-eth0
```

**ip地址：**172.17.0.4

**mac地址：**02:42:ac:11:00:04



# 二、tc规则整理

这里面有个请求方和应答方的概念。

## 2、1 vbr-neuv规则

```
/ # tc filter show dev vbr-neuv egress
## vbr-neuv -> vin5bcf-eth0流量镜像方向，匹配目的mac地址，如果匹配"4e:65:75:56:00:0d"，将目的mac改为02:42:ac:11:00:03(nginx 容器的mac地址)
filter parent ffff: protocol all pref 13 u32 chain 0 
filter parent ffff: protocol all pref 13 u32 chain 0 fh 802: ht divisor 1 
filter parent ffff: protocol all pref 13 u32 chain 0 fh 802::800 order 2048 key ht 802 bkt 0 terminal flowid ??? not_in_hw 
  match 00004e65/0000ffff at -16 ###匹配目的mac地址
  match 7556000d/ffffffff at -12
	action order 1:  pedit action pipe keys 2
 	 index 7 ref 1 bind 1
	 key #0  at -16: val 00000242 mask ffff0000
	 key #1  at -12: val ac110003 mask 00000000

	action order 2: mirred (Egress Mirror to device vin5bcf-eth0) pipe
	index 11 ref 1 bind 1

## vbr-neuv -> vex5bcf-eth0流量镜像方向，匹配源mac地址，如果匹配"4e:65:75:56:00:0d"，将源mac改为02:42:ac:11:00:03(nginx容器的mac地址)
filter parent ffff: protocol all pref 14 u32 chain 0 
filter parent ffff: protocol all pref 14 u32 chain 0 fh 803: ht divisor 1 
filter parent ffff: protocol all pref 14 u32 chain 0 fh 803::800 order 2048 key ht 803 bkt 0 terminal flowid ??? not_in_hw 
  match 4e657556/ffffffff at -8
  match 000d0000/ffff0000 at -4
	action order 1:  pedit action pipe keys 2
 	 index 8 ref 1 bind 1
	 key #0  at -8: val 0242ac11 mask 00000000
	 key #1  at -4: val 00030000 mask 0000ffff

	action order 2: mirred (Egress Mirror to device vex5bcf-eth0) pipe
	index 12 ref 1 bind 1

## vbr-neuv -> vin6514-eth0流量镜像方向，匹配目的mac地址，如果匹配"4e:65:75:56:00:13"，将目的mac改为02:42:ac:11:00:04(centos容器的mac地址)
filter parent ffff: protocol all pref 19 u32 chain 0 
filter parent ffff: protocol all pref 19 u32 chain 0 fh 800: ht divisor 1 
filter parent ffff: protocol all pref 19 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw 
  match 00004e65/0000ffff at -16 ###匹配目的mac地址
  match 75560013/ffffffff at -12
	action order 1:  pedit action pipe keys 2
 	 index 3 ref 1 bind 1
	 key #0  at -16: val 00000242 mask ffff0000
	 key #1  at -12: val ac110004 mask 00000000

	action order 2: mirred (Egress Mirror to device vin6514-eth0) pipe
	index 5 ref 1 bind 1

## vbr-neuv -> vex6514-eth0流量镜像方向，匹配源mac地址，如果匹配"4e:65:75:56:00:13"，将源mac改为02:42:ac:11:00:04(centos容器的mac地址)
filter parent ffff: protocol all pref 20 u32 chain 0 
filter parent ffff: protocol all pref 20 u32 chain 0 fh 801: ht divisor 1 
filter parent ffff: protocol all pref 20 u32 chain 0 fh 801::800 order 2048 key ht 801 bkt 0 terminal flowid ??? not_in_hw 
  match 4e657556/ffffffff at -8
  match 00130000/ffff0000 at -4
	action order 1:  pedit action pipe keys 2
 	 index 4 ref 1 bind 1
	 key #0  at -8: val 0242ac11 mask 00000000
	 key #1  at -4: val 00040000 mask 0000ffff

	action order 2: mirred (Egress Mirror to device vex6514-eth0) pipe
	index 6 ref 1 bind 1

```



## 2、2 vth-neuv规则

无规则，vth-neuv口是dp程序的抓包口。

vth-neuv在代码中是如何使用的呢？



## 2、3 vex5bcf-eth0和vin5bcf-eth0规则

### 2、3、1 vin5bcf-eth0

```
/ # tc filter show dev vin5bcf-eth0 egress
## vin5bcf-eth0 -> vbr-neuv流量镜像方向，匹配源mac地址，如果匹配"02:42:ac:11:00:03"，将源mac改为4e:65:75:56:00:0d
filter parent ffff: protocol ip pref 10001 u32 chain 0 
filter parent ffff: protocol ip pref 10001 u32 chain 0 fh 800: ht divisor 1
filter parent ffff: protocol ip pref 10001 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw 
  match 00000000/00000100 at -16
  match 0242ac11/ffffffff at -8
  match 00030000/ffff0000 at -4
	action order 1:  pedit action pipe keys 2
 	 index 6 ref 1 bind 1
	 key #0  at -8: val 4e657556 mask 00000000
	 key #1  at -4: val 000d0000 mask 0000ffff

	action order 2: mirred (Egress Mirror to device vbr-neuv) pipe
	index 9 ref 1 bind 1

## vin5bcf-eth0 -> vex5bcf-eth0流量镜像方向，剩余流量都走这个
filter parent ffff: protocol all pref 10002 u32 chain 0 
filter parent ffff: protocol all pref 10002 u32 chain 0 fh 801: ht divisor 1 
filter parent ffff: protocol all pref 10002 u32 chain 0 fh 801::800 order 2048 key ht 801 bkt 0 terminal flowid ??? not_in_hw 
  match 00000000/00000000 at 0
	action order 1: mirred (Egress Mirror to device vex5bcf-eth0) pipe
	index 10 ref 1 bind 1
```



### 2、3、2 vex5bcf-eth0

```
/ # tc filter show dev vex5bcf-eth0 egress
## vex5bcf-eth0 -> vbr-neuv流量镜像方向，匹配目的mac地址，如果匹配"02:42:ac:11:00:03"，将目的mac改为4e:65:75:56:00:0d
filter parent ffff: protocol ip pref 10001 u32 chain 0 
filter parent ffff: protocol ip pref 10001 u32 chain 0 fh 800: ht divisor 1 
filter parent ffff: protocol ip pref 10001 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw 
  match 00000242/0000ffff at -16
  match ac110003/ffffffff at -12
	action order 1:  pedit action pipe keys 2
 	 index 5 ref 1 bind 1
	 key #0  at -16: val 00004e65 mask ffff0000
	 key #1  at -12: val 7556000d mask 00000000

	action order 2: mirred (Egress Mirror to device vbr-neuv) pipe
	index 7 ref 1 bind 1

## vex5bcf-eth0 -> vin5bcf-eth0流量镜像方向，剩余流量都走这个
filter parent ffff: protocol all pref 10002 u32 chain 0 
filter parent ffff: protocol all pref 10002 u32 chain 0 fh 801: ht divisor 1 
filter parent ffff: protocol all pref 10002 u32 chain 0 fh 801::800 order 2048 key ht 801 bkt 0 terminal flowid ??? not_in_hw 
  match 00000000/00000000 at 0
	action order 1: mirred (Egress Mirror to device vin5bcf-eth0) pipe
	index 8 ref 1 bind 1
```



## 2、4 vex6514-eth0和vin6514-eth0规则

### 2、4、1 vin6514-eth0

```
/ # tc filter show dev vin6514-eth0 egress
## vin6514-eth0 -> vbr-neuv流量镜像方向，匹配源mac地址，如果匹配"02:42:ac:11:00:04"，将源mac改为4e:65:75:56:00:13
filter parent ffff: protocol ip pref 10001 u32 chain 0 
filter parent ffff: protocol ip pref 10001 u32 chain 0 fh 800: ht divisor 1 
filter parent ffff: protocol ip pref 10001 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw 
  match 00000000/00000100 at -16
  match 0242ac11/ffffffff at -8
  match 00040000/ffff0000 at -4
	action order 1:  pedit action pipe keys 2
 	 index 2 ref 1 bind 1
	 key #0  at -8: val 4e657556 mask 00000000
	 key #1  at -4: val 00130000 mask 0000ffff

	action order 2: mirred (Egress Mirror to device vbr-neuv) pipe
	index 3 ref 1 bind 1

## vin6514-eth0 -> vex6514-eth0流量镜像方向，剩余流量都走这个
filter parent ffff: protocol all pref 10002 u32 chain 0 
filter parent ffff: protocol all pref 10002 u32 chain 0 fh 801: ht divisor 1 
filter parent ffff: protocol all pref 10002 u32 chain 0 fh 801::800 order 2048 key ht 801 bkt 0 terminal flowid ??? not_in_hw 
  match 00000000/00000000 at 0
	action order 1: mirred (Egress Mirror to device vex6514-eth0) pipe
	index 4 ref 1 bind 1
```



### 2、4、2 vex6514-eth0

```
/ # tc filter show dev vex6514-eth0 egress
## vex6514-eth0 -> vbr-neuv流量镜像方向，匹配目的mac地址，如果匹配"02:42:ac:11:00:04"，将目的mac改为4e:65:75:56:00:13
filter parent ffff: protocol ip pref 10001 u32 chain 0 
filter parent ffff: protocol ip pref 10001 u32 chain 0 fh 800: ht divisor 1 
filter parent ffff: protocol ip pref 10001 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw 
  match 00000242/0000ffff at -16
  match ac110004/ffffffff at -12
	action order 1:  pedit action pipe keys 2
 	 index 1 ref 1 bind 1
	 key #0  at -16: val 00004e65 mask ffff0000
	 key #1  at -12: val 75560013 mask 00000000

	action order 2: mirred (Egress Mirror to device vbr-neuv) pipe
	index 1 ref 1 bind 1

## vex6514-eth0 -> vin6514-eth0流量镜像方向，剩余流量都走这个
filter parent ffff: protocol all pref 10002 u32 chain 0 
filter parent ffff: protocol all pref 10002 u32 chain 0 fh 801: ht divisor 1 
filter parent ffff: protocol all pref 10002 u32 chain 0 fh 801::800 order 2048 key ht 801 bkt 0 terminal flowid ??? not_in_hw 
  match 00000000/00000000 at 0
	action order 1: mirred (Egress Mirror to device vin6514-eth0) pipe
	index 2 ref 1 bind 1
```



其中频繁出现了两个mac地址

```
4e:65:75:56:00:0d
4e:65:75:56:00:13
```

这两个地址是unicast 地址-组播地址。



在普通模式下，并没有生成iptables规则，所以我们还是能处理好这个事情。



#todo servicemesh场景

servicemesh场景下的规则，

```
tc filter show dev vbr-neuv egress
filter parent ffff: protocol all pref 5 u32 chain 0
filter parent ffff: protocol all pref 5 u32 chain 0 fh 800: ht divisor 1
filter parent ffff: protocol all pref 5 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw
  match 00004e65/0000ffff at -16
  match 75560005/ffffffff at -12
        action order 1:  pedit action pipe keys 2
         index 3 ref 1 bind 1
         key #0  at -16: val 00000a3c mask ffff0000
         key #1  at -12: val 8cf9a266 mask 00000000

        action order 2: mirred (Egress Mirror to device vin74f9-eth0) pipe
        index 5 ref 1 bind 1

filter parent ffff: protocol all pref 27 u32 chain 0
filter parent ffff: protocol all pref 27 u32 chain 0 fh 801: ht divisor 1
filter parent ffff: protocol all pref 27 u32 chain 0 fh 801::800 order 2048 key ht 801 bkt 0 terminal flowid ??? not_in_hw
  match 4e657556/ffffffff at -8
  match 00050000/ffff0000 at -4
        action order 1:  pedit action pipe keys 2
         index 4 ref 1 bind 1
         key #0  at -8: val 0a3c8cf9 mask 00000000
         key #1  at -4: val a2660000 mask 0000ffff

        action order 2: mirred (Egress Mirror to device vex74f9-eth0) pipe
        index 6 ref 1 bind 1
```



```
tc filter show dev vbr-neuv ingress
filter parent ffff: protocol all pref 5 u32 chain 0
filter parent ffff: protocol all pref 5 u32 chain 0 fh 800: ht divisor 1
filter parent ffff: protocol all pref 5 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw
  match 00004e65/0000ffff at -16
  match 75560005/ffffffff at -12
        action order 1:  pedit action pipe keys 2
         index 3 ref 1 bind 1
         key #0  at -16: val 00000a3c mask ffff0000
         key #1  at -12: val 8cf9a266 mask 00000000

        action order 2: mirred (Egress Mirror to device vin74f9-eth0) pipe
        index 5 ref 1 bind 1

filter parent ffff: protocol all pref 27 u32 chain 0
filter parent ffff: protocol all pref 27 u32 chain 0 fh 801: ht divisor 1
filter parent ffff: protocol all pref 27 u32 chain 0 fh 801::800 order 2048 key ht 801 bkt 0 terminal flowid ??? not_in_hw
  match 4e657556/ffffffff at -8
  match 00050000/ffff0000 at -4
        action order 1:  pedit action pipe keys 2
         index 4 ref 1 bind 1
         key #0  at -8: val 0a3c8cf9 mask 00000000
         key #1  at -4: val a2660000 mask 0000ffff

        action order 2: mirred (Egress Mirror to device vex74f9-eth0) pipe
        index 6 ref 1 bind 1
```

