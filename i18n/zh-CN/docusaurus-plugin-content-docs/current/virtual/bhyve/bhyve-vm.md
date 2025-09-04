---
sidebar_position: 2
---

# 在 bhyve 中运行 虚拟机

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

[Github: freebsd/vm-bhyve](https://github.com/freebsd/vm-bhyve) 是一个基于 Shell 的最小化依赖的 bhyve manager ，非常方便管理虚拟机，强烈推荐。(我之前直接使用 `bhyve` 需要使用大量命令参数，非常麻烦；通过 `vm-bhyve` 管理则简化了操作)

- 安装 `vm-bhyve` :

```bash
pkg install vm-bhyve bhyve-firmware
```

- 为 `vm-bhyve` 准备存储目录(采用 `zroot/vms` 的 [ZFS](../../os/freebsd/freebsd-zfs) )

- `vm-bhyve` 需要指定存储目录，并且需要激活，所以配置 `/etc/rc.conf`

```bash
# needed for virtualization support
vm_enable="YES"
vm_dir="zfs:zroot/vms"
```

:::note

我在 [ZFS的zroot存储池为vm-bhyve配置了独立的dataset](../../os/freebsd/freebsd-zfs) `zroot/vms` ，所以上面配置中指定 `vm_dir`

:::

- 在 `/boot/loader.conf` 添加:

```bash
# needed for virtualization support 
vmm_load="YES"
```

- 初始化 `vm-bhyve` :

```bash
vm init
```

- 在 `/usr/local/share/examples/vm-bhyve/` 目录下有虚拟机的配置模版案例，所谓的配置模版其实是一些简单的配置项来指定虚拟机如何运行。对于 `vm-bhyve` 使用的模版是存放在 `$vm_dir/.templates` ，所以执行以下命令复制模版:

```bash
cp /usr/local/share/examples/vm-bhyve/* /zroot/vms/.templates/
```

:::note

复制模版文件中有一个 `config.sample` 包含了详细的配置解析，可以参考这个配置案例来修订自己的配置文件

:::

:::tip

实际上比较简洁的方式是参考 `config.sample` 配置自己定制的配置，例如我的网络配置和 `default` 不同

:::

- 默认情况下可以创建一个 `public` 虚拟机交换机，然后将物理主机的某个网卡加入(例如 `em0` )，这样后续配置虚拟机的时候，只要为虚拟机指定这个 `public` 虚拟机交换机，那么虚拟机的虚拟网卡就会连接上 `public` 虚拟交换机，并通过 `em0` 网卡连接外部物理真实网络:

```bash
vm switch create public
vm switch add public em0
```

:::note

我的实践有所不同:

- 我已经在 [FreeBSD初始化](../../os/freebsd/freebsd-init) 中配置了一个名为 `igc0bridge` 的虚拟交换机，连接了物理网络
- [bhyve虚拟化](../../virtual/bhyve/bhyve-blueprint) 和 [Jails容器](../../container/jails/jails-blueprint) 将共用这个 `igc0bridge` 虚拟交换机

:::

### 创建Ubuntu虚拟机

- 创建自定义模版文件 `/zroot/vms/.templates/x-vm.conf`

```bash
loader="uefi"
cpu=4
memory=16G
wired_memory="yes"
network0_type="virtio-net"
network0_switch="igc0bridge"
network0_device="tap0"
disk0_name="disk0"
disk0_dev="sparse-zvol"
disk0_type="virtio-blk"
disk0_size="50G"
#passthru0="1/0/0=2:0"
graphics="yes"
graphics_listen="0.0.0.0"
graphics_port="5900"
```

:::note

- `wired_memory` 是因为 [在bhyve实现NVIDIA PCI Passthrough](bhyve-nvidia-gpu-passthru) 需要

- 配置中指定 `disk0_dev="sparse-zvol"` :

  - 对于host主机具备 [ZFS](../../os/freebsd/freebsd-zfs) 环境， **使用ZFS volumes来代替磁盘镜像，可以获得明显的性能提升** ，所以配置中指定 `disk0_dev="sparse-zvol"`
  - 参数创建稀疏卷(sparse volume)，raw disk不是马上分配，而是在数据写入时动态分配(可以分配远大于物理磁盘空间的虚拟磁盘，必要时再扩容底层zfs存储)
  -  `vm-bhyve` 会自动创建 `/zroot/vms/<虚拟机名>` 作为虚拟机存储数据集，并在下面创建一个 `sparse volume` raw磁盘 `disk0`

- 配置中指定 `network0_switch="igc0bridge"` : 预先配置的 `igc0bridge` 虚拟交换机

- 配置中指定 `network0_device="tap0"` : 设置了 `tap0` 而不是让 `vm-bhyve` 自动生成

  - 我发现我配置了指定交换机之后，自动生成的 `tap` 设备没有 `addm` 到我指定的 `igc0bridge` 虚拟交换机，导致网络不通
  - 参考 `config.sample` 指定了我之前配置好的 `tap0` (已经连接在 `igc0bridge` 上)解决了这个网络问题

- 配置中指定 `disk0_type="virtio-blk"` :

  - `virtio` 是 `bhyve` 内置支持的设备，提供了 para-virtualization 高性能

:::

- 创建虚拟机

```bash
vm create -t x-vm -s 60G xdev
```

:::note

- `-t x-vm` 表示使用 `/zroot/vms/.templates` 目录下的 `x-vm.conf` 作为模版
- `-s 60G` 创建了一个特定的 `60G` 虚拟磁盘，覆盖了模版中配置 `50G`

:::

- 下载Ubuntu安装镜像(自动保存到 `/zroot/.iso/` 目录下)

```bash
vm iso https://mirrors.jxust.edu.cn/ubuntu-releases/24.04.3/ubuntu-24.04.3-live-server-amd64.iso
```

- 安装ubuntu:

```bash

vm install xdev ubuntu-24.04.3-live-server-amd64.iso
```

此时虚拟机安装会启动VNC服务监听在5900端口，通过VNC客户端访问并继续安装

## 进一步阅读

- [bhyve虚拟化运行Ubuntu](https://docs.cloud-atlas.dev/discovery/freebsd/virtual/bhyve/bhyve_ubuntu.html)
- [vm-bhyve](https://docs.cloud-atlas.dev/discovery/freebsd/virtual/bhyve/vm-bhyve.html)
