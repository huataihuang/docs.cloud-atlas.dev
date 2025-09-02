---
sidebar_position: 2
---

# 在 bhyve 中运行 ubuntu

为机器学习准备环境

## 主机准备工作

### 内核模块

- 加载 `bhyve` 内核模块:

```bash
kldload vmm
kldload nmdm
kldload if_tap
kldload if_bridge
```

- 设置启动时加载内核模块:

```bash
echo 'vmm_load="YES"' >> /boot/loader.conf
echo 'nmdm_load="YES"' >> /boot/loader.conf
echo 'if_tap_load="YES"' >> /boot/loader.conf
echo 'if_bridge_load="YES"' >> /boot/loader.conf
```

### 创建网桥和tap

- 在 `/etc/sysctl.conf` 中配置tap设备在操作系统启动时启动 

```bash
echo "net.link.tap.up_on_open=1" >> /etc/sysctl.conf
sysctl net.link.tap.up_on_open=1
```

- 创建网桥并连接网卡( 案例中使用的网卡设备是 `igc0` ): 

  - 已经在 [FreeBSD初始化](../../os/freebsd/freebsd-init) 时完成创建了 `igc0bridge`
  - 已启用了3个 `tap` 设备用于后续虚拟机连接

### 安装 `vm-bhyve`

[Github: freebsd/vm-bhyve](https://github.com/freebsd/vm-bhyve) 是一个基于 Shell 的最小化依赖的 bhyve manager ，非常方便管理虚拟机，强烈推荐。

- 安装 `vm-bhyve` :

```bash
pkg install vm-bhyve bhyve-firmware
```

## 进一步阅读

- [bhyve虚拟化运行Ubuntu](https://docs.cloud-atlas.dev/discovery/freebsd/virtual/bhyve/bhyve_ubuntu.html)
- [vm-bhyve](https://docs.cloud-atlas.dev/discovery/freebsd/virtual/bhyve/vm-bhyve.html)
