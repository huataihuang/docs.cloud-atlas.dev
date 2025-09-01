---
sidebar_position: 3
---

# FreeBSD 初始化

分不同环境(Host主机/VM/Jail)对FreeBSD系统进行初始化安装

## Host主机

### 安装工具软件

- 我在 [组装台式机](../../hardware/assembly-machine) 上安装最基本的FreeBSD，并努力保持Host主机轻量化(只安装最基本维护工具)，所有的日常工作都在虚拟机或Jail中完成

```bash
# tmux 建议同时安装 terminfo-db 以获取terminfo数据库
pkg install sudo tmux terminfo-db
```

### 虚拟交换机

[组装台式机](../../hardware/assembly-machine) 主板有4个2.5GB以太网卡，为了能够充分发挥硬件，我构建了一个 `bridge` 来连接3台 [树莓派](../../hardware/raspberry-pi) `5` 以及上联交换机:

- 修订 `/boot/loader.conf` 加载 `if_bridge` 模块(可选加载 `bridgestp` )

```bash
if_bridge_load="YES"
#bridgestp_load="YES"
```

- 修改 `/etc/rc.conf` 配置交换机以及将物理和虚拟网卡连接(详情参考 [FreeBSD bridge快速起步](https://docs.cloud-atlas.dev/discovery/freebsd/network/bridge/freebsd_bridge_startup.html)

```bash
# 创建网桥以及用于虚拟机的tap虚拟网络设备
cloned_interfaces="bridge0 tap0 tap1 tap2"
# 重命名网桥
ifconfig_bridge0_name="igc0bridge"
# 将物理网卡(4个igcX)和虚拟网卡(tapX)连接到网桥上
ifconfig_igc0bridge="inet 192.168.7.200/24 addm igc0 addm igc1 addm igc2 addm igc3 addm tap0 addm tap1 addm tap2 up"
# 设置默认网关
defaultrouter="192.168.7.221"
# 激活物理网卡
ifconfig_igc0="up"
ifconfig_igc1="up"
ifconfig_igc2="up"
ifconfig_igc3="up"
```

完成上述配置后，重启系统确保(重启)生效

## 虚拟机

## Jail
