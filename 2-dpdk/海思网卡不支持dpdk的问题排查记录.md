# 一、现场问题描述

客户现场的服务器的网卡是海思的网卡，但是在dpdk初始化时报错，

![image-20250731102320337](https://gitee.com/codergeek/picgo-image/raw/master/image/202507311023745.png)

首先梳理下报错堆栈如下

hns3_dev_init -> hns3_init_pf -> hns3_get_configuration -> hns3_check_media_type

hns3_check_media_type函数中报错Media type is copper，not supported.

# 二、问题初步分析

## 2、1 问题追码溯源

在dpdk源代码中搜索报错信息"Media type is copper，not supported"，如下源代码所示：

```
static int
hns3_check_media_type(struct hns3_hw *hw, uint8_t media_type)
{
	int ret;

	switch (media_type) {
	case HNS3_MEDIA_TYPE_COPPER:
		//检测到介质为铜缆类型后
		//调用hns3_dev_get_support检查硬件是否支持COPPER功能
		if (!hns3_dev_get_support(hw, COPPER)) {
			PMD_INIT_LOG(ERR,
				     "Media type is copper, not supported.");//出错信息在这里！！！
			ret = -EOPNOTSUPP;
		} else {
			ret = 0;
		}
		break;
	case HNS3_MEDIA_TYPE_FIBER:
	case HNS3_MEDIA_TYPE_BACKPLANE://介质为背板
		ret = 0;
		break;
	default:
		PMD_INIT_LOG(ERR, "Unknown media type = %u!", media_type);//检测到为止的介质类型
		ret = -EINVAL;
		break;
	}

	return ret;
}
```

海思网卡的介质类型分为三种：

- **铜质电缆：**使用传统的铜质网线（如Cat5e、Cat6、Cat7等），支持电信号传输，需要PHY芯片来处理电信号
- **光纤：**支持光信号传输，支持更高的带宽和更低的延迟
- **背板：**在设备内部通过背板连接，支持高速内部通信

hns3_dev_get_support是一个宏定义，其细节如下：

```
#define hns3_dev_get_support(hw, _name) \
	hns3_get_bit((hw)->capability, HNS3_DEV_SUPPORT_##_name##_B)
```

代码转换后为

```
hns3_get_bit((hw)->capability, HNS3_DEV_SUPPORT_COPPER_B)
```

即查看网卡能力capability字段是否支持HNS3_DEV_SUPPORT_COPPER_B。

在dpdk的24.11.2版本的代码中，设置网卡能力capability字段的标志位（我们关心的是HNS3_DEV_SUPPORT_COPPER_B标志位）：

```
static void
hns3_parse_capability(struct hns3_hw *hw,
		      struct hns3_query_version_cmd *cmd)
{
	uint32_t caps = rte_le_to_cpu_32(cmd->caps[0]);

	...设置网卡的其他能力标识...
	//校验标志位，并设置网卡的HNS3_DEV_SUPPORT_COPPER_B标识
	if (hns3_get_bit(caps, HNS3_CAPS_PHY_IMP_B))
		hns3_set_bit(hw->capability, HNS3_DEV_SUPPORT_COPPER_B, 1);
	/* Force enable copper support for compatibility */
	//强行设置支持铜缆介质的能力
	//在24.11.1添加了这行代码以提升兼容性
	//但是在21.11.2 LTS稳定版的代码中又把这行代码删除了，哈哈，笑死我
	hns3_set_bit(hw->capability, HNS3_DEV_SUPPORT_COPPER_B, 1);
	...设置网卡的其他能力标识...
}
```

初步看这个问题，是因为HNS3_DEV_SUPPORT_COPPER_B标志没有成功设置到hw->capability变量中导致的问题。



## 2、2 官网patch修复

在官网查找这个问题的相关mailist，发现官网给出过修复方案，https://inbox.dpdk.org/stable/20250218123523.36836-73-xuemingl@nvidia.com/

我还特意下载了dpdk 25.07的代码进行对比，修复后的代码如下：

```
static int hns3_set_port_link_speed(struct hns3_hw *hw,
			 struct hns3_set_link_speed_cfg *cfg)
{
	int ret;

	//介质为copper铜缆时，调用hns3_set_copper_port_link_speed
	if (hw->mac.media_type == HNS3_MEDIA_TYPE_COPPER)
		ret = hns3_set_copper_port_link_speed(hw, cfg);
	else
		ret = hns3_set_fiber_port_link_speed(hw, cfg);

	if (ret) {
		hns3_err(hw, "failed to set %s port link speed, ret = %d.",
			 hns3_get_media_type_name(hw->mac.media_type),
			 ret);
		return ret;
	}

	return 0;
}
```

当介质类型为copper铜缆时，调用hns3_set_copper_port_link_speed进行设置相关网卡属性。

当介质类型为fiber光纤时，调用hns3_set_fiber_port_link_speed进行设置相关网卡属性。

hns3_set_copper_port_link_speed函数的定义如下所示：

```
static int hns3_set_copper_port_link_speed(struct hns3_hw *hw,
				struct hns3_set_link_speed_cfg *cfg)
{
#define HNS3_PHY_PARAM_CFG_RETRY_TIMES		10
#define HNS3_PHY_PARAM_CFG_RETRY_DELAY_MS	100
	uint32_t retry_cnt = 0;
	int ret;

	/*
	 * The initialization of copper port contains the following two steps.
	 * 1. Configure firmware takeover the PHY. The firmware will start an
	 *    asynchronous task to initialize the PHY chip.
	 * 2. Configure work speed and duplex.
	 * In earlier versions of the firmware, when the asynchronous task is not
	 * finished, the firmware will return -ENOTBLK in the second step. And this
	 * will lead to driver failed to initialize. Here add retry for this case.
	 */
	ret = hns3_copper_port_link_speed_cfg(hw, cfg);
	while (ret == -ENOTBLK && retry_cnt++ < HNS3_PHY_PARAM_CFG_RETRY_TIMES) {
		rte_delay_ms(HNS3_PHY_PARAM_CFG_RETRY_DELAY_MS);
		ret = hns3_copper_port_link_speed_cfg(hw, cfg);
	}

	return ret;
}
```

这个函数会循环调用hns3_copper_port_link_speed_cfg来设置端口链路速率的配置信息，主要的修复工作就是这里的while循环操作。为什么呢？

铜缆初始化包括以下两个步骤：

1、配置固件接管PHY。固件将启动一个异步任务来初始化PHY芯片。

2、配置工作速率和双工模式

在早期版本的固件中，当异步任务尚未完成时，固件会在第二步返回-ENOTBLK错误。

这将导致驱动程序初始化失败，解决方案是添加了错误重试机制，重试10次。



# 三、解决方案

**方案1：**将dpdk从24.11.2版本升级到最新版本（比如25.07，其他版本也可以）

**方案2：**升级固件版本，固件版本过低是导致这个问题的根本原因。



