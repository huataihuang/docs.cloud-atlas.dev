---
sidebar_position: 2
---

# VNET + Thin Jail

构建 VNET + Thin Jails

## 主机激活 jail

- 执行以下命令配置在系统启动时启动 ``Jail`` :

```bash
# 配置在系统启动时激活Jail功能
sysrc jail_enable="YES"

# 配置所有jails在后台启动
sysrc jail_parallel_start="YES"
```

## Jail目录树

- 我使用 [stripe 模式 ZFS](../../os/freebsd/freebsd-zfs#创建-stripe-模式-zfs-pool-zdata) 存储池 `zdata` 下的 `/zdata/jails` 卷来作为Jail文件存储:

```bash
zfs create zdata/jails
zfs create zdata/jails/media
zfs create zdata/jails/templates
zfs create zdata/jails/containers
```

完成后检查 ``df -h`` 可以看到磁盘如下:

```
Filesystem                   Size    Used   Avail Capacity  Mounted on
...
zdata                        6.1T     96K    6.1T     0%    /zdata
zdata/jails                  6.1T    104K    6.1T     0%    /zdata/jails
zdata/jails/media            6.1T     96K    6.1T     0%    /zdata/jails/media
zdata/jails/templates        6.1T     96K    6.1T     0%    /zdata/jails/templates
zdata/jails/containers       6.1T     96K    6.1T     0%    /zdata/jails/containers
```

:::tip
- `media` 将包含已下载用户空间的压缩文件
- `templates` 在使用 Thin Jails 时，该目录存储模板(共享核心系统)
- `containers` 将存储jail (也就是容器)
:::

## Thin(薄) Jail

### `快照(snapshot)` 型 *vs.* `模板和NullFS` 型

FreeBSD Thin Jail是基于 [ZFS](../../os/freebsd/freebsd-zfs) `快照(snapshot)` 或 `模板和NullFS` 来创建的 瘦 Jail:

- 使用区别:

  - `快照(snapshot)` 型: 快照只读的，不可更改的 - 这意味着 **没有办法简单通过更新共享的ZFS snapshot来实现Thin Jail操作系统更新**
  - `模板和NullFS` 型: 可以通过 **直接更新NullFS底座共享的ZFS dataset可以瞬间更新所有Thin Jail**

- 安装区别:

  - ZFS snapshot Thin Jail: 将FreeBSD Release base存放在 **只读** 的 `14.2-RELEASE@base` 快照上
  - NullFS Thin Jail: 将FreeBSD Release base存放在 **读写** 的 `14.2-RELEASE-base` 数据集上

### 我的选择: `模板和NullFS`

通过结合Thin Jail 和 `NullFS` 技术可以创建节约文件系统存储开销(类似于 ZFS `snapshot` clone出来的卷完全不消耗空间)，并且能够将Host主机的目录共享给 **多个** Jail。

- 创建 **读写模式** 的 `14.2-RELEASE-base` (注意，大家约定俗成 `@base` 表示只读快照， `-base` 表示可读写数据集)

```bash
zfs create -p zdata/jails/templates/14.2-RELEASE-base
```

- 下载用户空间:

```bash
fetch https://download.freebsd.org/ftp/releases/amd64/amd64/14.2-RELEASE/base.txz -o /zdata/jails/media/14.2-RELEASE-base.txz
```

- 将下载内容解压缩到模版目录: **内容解压缩到模板目录( 14.2-RELEASE-base 后续不需要创建快照，直接使用)**

```bash
tar -xf /zdata/jails/media/14.2-RELEASE-base.txz -C /zdata/jails/templates/14.2-RELEASE-base --unlink
```

- 将时区和DNS配置复制到模板目录:

```bash
cp /etc/resolv.conf /zdata/jails/templates/14.2-RELEASE-base/etc/resolv.conf
cp /etc/localtime   /zdata/jails/templates/14.2-RELEASE-base/etc/localtime
```

- 更新模板补丁:

```bash
freebsd-update -b /usr/local/jails/templates/14.2-RELEASE-base/ fetch install
```

:::tip
关键步骤:以下是NullFS特别部分( `skeleton` )
:::

- 创建一个特定数据集 `skeleton` (**骨骼**) ，这个 "骨骼" `skeleton` 命名非常形象，用意就是构建特殊的支持大量thin jial的框架底座

```bash
zfs create -p zdata/jails/templates/14.2-RELEASE-skeleton
```

- 执行以下命令，将特定目录移入 `skeleton` 数据集，并构建 `base` 和 `skeleton` 必要目录的软连接关系

```bash
mkdir -p /zdata/jails/templates/14.2-RELEASE-skeleton/home
mkdir -p /zdata/jails/templates/14.2-RELEASE-skeleton/usr
# etc目录包含发行版提供的配置文件
mv /zdata/jails/templates/14.2-RELEASE-base/etc /zdata/jails/templates/14.2-RELEASE-skeleton/etc
# local目录是空的
mv /zdata/jails/templates/14.2-RELEASE-base/usr/local /zdata/jails/templates/14.2-RELEASE-skeleton/usr/local
# tmp 目录是空的
mv /zdata/jails/templates/14.2-RELEASE-base/tmp /zdata/jails/templates/14.2-RELEASE-skeleton/tmp
# var 目录有很多预存目录，其中移动时有报错显示 var/empty 目录没有权限
mv /zdata/jails/templates/14.2-RELEASE-base/var /zdata/jails/templates/14.2-RELEASE-skeleton/var
# root 目录是管理员目录，有基本profile文件
mv /zdata/jails/templates/14.2-RELEASE-base/root /zdata/jails/templates/14.2-RELEASE-skeleton/root
```

:::tip
执行 `mv /zdata/jails/templates/14.2-RELEASE-base/var /zdata/jails/templates/14.2-RELEASE-skeleton/var` 有如下报错:

```bash
mv: /zdata/jails/templates/14.2-RELEASE-base/var/empty: Operation not permitted
mv: /zdata/jails/templates/14.2-RELEASE-base/var: Directory not empty
mv: /bin/rm /zdata/jails/templates/14.2-RELEASE-base/var: terminated with 1 (non-zero) status
```

这是因为  `var/empty` 目录没有权限删除: `mv var/empty: Operation not permitted`

我采用以下workround绕过:

```bash
cd /zdata/jails/templates/14.2-RELEASE-base/
mv var var.bak
```
:::

- 执行以下命令创建软连接:

```bash
cd /zdata/jails/templates/14.2-RELEASE-base/
mkdir skeleton
ln -s skeleton/etc etc
ln -s skeleton/home home
ln -s skeleton/root root
ln -s skeleton/usr/local usr/local
ln -s skeleton/tmp tmp
ln -s skeleton/var var
```

- 在 `skeleton` 就绪之后，需要将数据复制到 jail 目录(如果是UFS文件系统)，对于ZFS则非常方便使用快照:

```bash
zfs snapshot zdata/jails/templates/14.2-RELEASE-skeleton@base
# 假设这里创建名为dev的jail
zfs clone zdata/jails/templates/14.2-RELEASE-skeleton@base zdata/jails/containers/dev
```

- 现在可以看到相关ZFS数据集如下:

```bash
Filesystem                                     Size    Used   Avail Capacity  Mounted on
zroot/ROOT/default                             192G    1.2G    191G     1%    /
...
zdata/jails                                    6.1T    112K    6.1T     0%    /zdata/jails
zdata/jails/media                              6.1T    197M    6.1T     0%    /zdata/jails/media
zdata/jails/templates                          6.1T    104K    6.1T     0%    /zdata/jails/templates
zdata/jails/containers                         6.1T     96K    6.1T     0%    /zdata/jails/containers
zdata/jails/templates/14.2-RELEASE-base        6.1T    446M    6.1T     0%    /zdata/jails/templates/14.2-RELEASE-base
zdata/jails/templates/14.2-RELEASE-skeleton    6.1T    4.4M    6.1T     0%    /zdata/jails/templates/14.2-RELEASE-skeleton
zdata/jails/containers/dev                     6.1T    4.4M    6.1T     0%    /zdata/jails/containers/dev
```

- 创建一个 `base` template的目录，这个目录是 `skeleton` 挂载所使用的根目录:

```bash
# 创建 dev 所使用的nullfs模板目录，这个目录就是jail的根PATH
# 和常规Jail仅命名 ${name} 不同，采用 ${name}-nullfs-base
mkdir -p /zdata/jails/dev-nullfs-base
```

### 配置jail

- 创建所有jail使用的公共配置部分 `/etc/jail.conf` :

```bash
# STARTUP/LOGGING
exec.start = "/bin/sh /etc/rc";
exec.stop = "/bin/sh /etc/rc.shutdown";
exec.consolelog = "/var/log/jail_console_${name}.log";

# PERMISSIONS
allow.raw_sockets;
exec.clean;
mount.devfs;

# HOSTNAME/PATH
host.hostname = "${name}";

# NETWORK
#ip4 = inherit;
interface = igc0bridge;

.include "/etc/jail.conf.d/*.conf";
```

- `/etc/jail.conf.d/dev.conf` 独立配置部分( 使用了 VNET 模式配置):

```bash
dev {
  # 这里 devfs_ruleset 和Linux Jail的4不同
  devfs_ruleset=5;
  # 去除了 ip4.addr 配置

  # HOSTNAME/PATH
  path = "/zdata/jails/${name}-nullfs-base";
 
  # VNET/VIMAGE
  vnet;
  vnet.interface = "${epair}b";

  # NETWORKS/INTERFACES
  $id = "253";
  $ip = "192.168.7.${id}/24";
  $gateway = "192.168.7.101";
  $bridge = "igc0bridge"; 
  $epair = "epair${id}";

  # ADD TO bridge INTERFACE
  exec.prestart += "ifconfig ${epair} create up";
  exec.prestart += "ifconfig ${epair}a up descr jail:${name}";
  exec.prestart += "ifconfig ${bridge} addm ${epair}a up";
  exec.start    += "ifconfig ${epair}b ${ip} up";
  exec.start    += "route add default ${gateway}";
  exec.poststop = "ifconfig ${bridge} deletem ${epair}a";
  exec.poststop += "ifconfig ${epair}a destroy";
  
  # MOUNT
  mount.fstab = "/zdata/jails/${name}-nullfs-base.fstab";
}
```

- 注意，这里配置引用了一个针对nullfs的fstab配置，所以还需要创建一个 `/zdata/jails/dev-nullfs-base.fstab` :

```bash
/zdata/jails/templates/14.2-RELEASE-base  /zdata/jails/dev-nullfs-base/         nullfs  ro  0 0
/zdata/jails/containers/dev               /zdata/jails/dev-nullfs-base/skeleton nullfs  rw  0 0
```

- 最后启动 `dev` :

```bash
service jail start dev
```

通过 `rexec dev` 进入jail

- 设置Jail `dev` 在操作系统启动时启动，修改 `/etc/rc.conf`

```bash
jail_enable="YES"
jail_parallel_start="YES"
jail_list="dev"
jail_reverse_stop="YES"
```
