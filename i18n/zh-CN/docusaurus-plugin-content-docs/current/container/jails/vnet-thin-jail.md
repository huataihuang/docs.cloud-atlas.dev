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

:::tip
Jail文件的位置没有规定，在FreeBSD Handbook中采用的是 `/usr/local/jails` 目录。

我为了方便管理和数据存储最大化，使用了 [stripe 模式 ZFS](../../os/freebsd/freebsd-zfs#创建-stripe-模式-zfs-pool-zdata) 存储池 `zdata` 下的 `/zdata/jails`

注意，我使用了 `jail_zfs` 环境变量来指定ZFS位置，对应目录就是 `/$jail_zfs`
:::

- 创建jail目录结构

```bash
export jail_zfs="zdata/jails"
export bsd_ver="14.2"
# 在FreeBSD中root用户的shell默认是sh，所以调整 ~/.shrc
echo 'jail_zfs="zdata/jails"' >> ~/.shrc
echo 'bsd_ver="14.2"' >> ~/.shrc

zfs create $jail_zfs
zfs create $jail_zfs/media
zfs create $jail_zfs/templates
zfs create $jail_zfs/containers
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
zfs create -p $jail_zfs/templates/$bsd_ver-RELEASE-base
```

- 下载用户空间:

```bash
fetch https://download.freebsd.org/ftp/releases/amd64/amd64/$bsd_ver-RELEASE/base.txz -o /$jail_zfs/media/$bsd_ver-RELEASE-base.txz
```

- 将下载内容解压缩到模版目录: **内容解压缩到模板目录( 14.2-RELEASE-base 后续不需要创建快照，直接使用)**

```bash
tar -xf /$jail_zfs/media/$bsd_ver-RELEASE-base.txz -C /$jail_zfs/templates/$bsd_ver-RELEASE-base --unlink
```

- 将时区和DNS配置复制到模板目录:

```bash
cp /etc/resolv.conf /$jail_zfs/templates/$bsd_ver-RELEASE-base/etc/resolv.conf
cp /etc/localtime   /$jail_zfs/templates/$bsd_ver-RELEASE-base/etc/localtime
```

- 更新模板补丁:

```bash
freebsd-update -b /$jail_zfs/templates/$bsd_ver-RELEASE-base/ fetch install
```

:::tip
关键步骤:以下是NullFS特别部分( `skeleton` )
:::

- 创建一个特定数据集 `skeleton` (**骨骼**) ，这个 "骨骼" `skeleton` 命名非常形象，用意就是构建特殊的支持大量thin jial的框架底座

```bash
zfs create -p $jail_zfs/templates/$bsd_ver-RELEASE-skeleton
```

- 执行以下命令，将特定目录移入 `skeleton` 数据集，并构建 `base` 和 `skeleton` 必要目录的软连接关系

```bash
mkdir -p /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/home
mkdir -p /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/usr
# etc目录包含发行版提供的配置文件
mv /$jail_zfs/templates/$bsd_ver-RELEASE-base/etc /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/etc
# local目录是空的
mv /$jail_zfs/templates/$bsd_ver-RELEASE-base/usr/local /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/usr/local
# tmp 目录是空的
mv /$jail_zfs/templates/$bsd_ver-RELEASE-base/tmp /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/tmp
# var 目录有很多预存目录，其中移动时有报错显示 var/empty 目录没有权限
mkdir /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/var
rsync -avz /$jail_zfs/templates/$bsd_ver-RELEASE-base/var/ /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/var
mv /$jail_zfs/templates/$bsd_ver-RELEASE-base/var /$jail_zfs/templates/$bsd_ver-RELEASE-base/var.bak
# root 目录是管理员目录，有基本profile文件
mv /$jail_zfs/templates/$bsd_ver-RELEASE-base/root /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/root
```

:::tip
执行 `mv /$jail_zfs/templates/$bsd_ver-RELEASE-base/var /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/var` 有如下报错:

```bash
mv: /zdata/jails/templates/14.2-RELEASE-base/var/empty: Operation not permitted
mv: /zdata/jails/templates/14.2-RELEASE-base/var: Directory not empty
mv: /bin/rm /zdata/jails/templates/14.2-RELEASE-base/var: terminated with 1 (non-zero) status
```

这是因为  `var/empty` 目录没有权限删除: `mv var/empty: Operation not permitted` 

**警告⚠️，上述mv命令会导致源和目标var目录都残缺损坏，所以千万不要执行，而应该采用rsyc同步目录**

所以 **必须** 先复制目录，也就是使用 ``rsync`` ，然后再移除原var目录
:::

- 执行以下命令创建软连接:

```bash
cd /$jail_zfs/templates/$bsd_ver-RELEASE-base/
mkdir skeleton
ln -s skeleton/etc etc
ln -s skeleton/home home
ln -s skeleton/root root
ln -s ../skeleton/usr/local usr/local
ln -s skeleton/tmp tmp
ln -s skeleton/var var
```

:::warning
我在实践NullFS的Thin Jails，发现移动到 `skeleton` 目录下的 `/etc` 目录有一个子目录 `/etc/ssl/certs` 。这个证书目录下的文件是软链接到 `../../../usr/share/certs/trusted/` 目录下的证书文件。由于 `/etc` 目录移动后会导致这些相对链接失效，所以需要有一个修复软链接的步骤。

如果不执行这个证书软链接修复，则后续host主机上执行 `pkg -j <jail_name> install <package_name>` 会报错；而在jail中执行 `pkg` 命令会显示证书相关错误:

```
Certificate verification failed for /C=US/O=Let's Encrypt/CN=E6
```
:::

- 修复 `/etc/ssl/certs` 目录下证书文件软链接:

```bash
cd /$jail_zfs/templates/$bsd_ver-RELEASE-skeleton/etc/ssl/certs

# 先保存一份原始列表记录
ls -lh > /tmp/fix_link.txt

# 生成unlink命令
ls | sed "s@^@unlink @" > /tmp/unlink.sh

# 生成fix命令
# 这里不能使用绝对路径链接，否则会报错 link: ffdd40f9.0: Cross-device link
# ls -lh | awk '{print $NF, $(NF-2)}' | cut -c 9- | tail -n +2 | sed  "s@^@link @" > /tmp/fix_link.sh
ls -lh | awk '{print $NF, $(NF-2)}' | tail -n +2 | sed 's@^@ln -s ../@' > /tmp/fix_link.sh

# 检查一下 /tmp/fix_link.sh 是否满足要求
# 没有问题在执行以下2条命令

sh /tmp/unlink.sh
sh /tmp/fix_link.sh
```

- 在 `skeleton` 就绪之后，需要将数据复制到 jail 目录(如果是UFS文件系统)，对于ZFS则非常方便使用快照:

```bash
zfs snapshot $jail_zfs/templates/$bsd_ver-RELEASE-skeleton@base
# 假设这里创建名为dev的jail
zfs clone $jail_zfs/templates/$bsd_ver-RELEASE-skeleton@base $jail_zfs/containers/dev
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
mkdir -p /$jail_zfs/dev-nullfs-base
```

### 配置jail

:::tip
- Jail的配置分为公共部分和特定部分，公共部分涵盖了所有jails共有的配置
- 尽可能提炼出Jails的公共部分，这样就可以简化针对每个jail的特定部分，方便编写较稳维护
:::

- 创建所有jail使用的公共配置部分 `/etc/jail.conf` (使用了 VNET 模式配置):

```bash
 # 这里 devfs_ruleset 和Linux Jail的4不同
 devfs_ruleset=5;

# STARTUP/LOGGING
exec.start = "/bin/sh /etc/rc";
exec.stop = "/bin/sh /etc/rc.shutdown";
exec.consolelog = "/var/log/jail_console_${name}.log";

# PERMISSIONS
allow.raw_sockets;
exec.clean;
mount.devfs;

# HOSTNAME/PATH - NullFS
host.hostname = "${name}";
path = "/zdata/jails/${name}-nullfs-base";

# NETWORK - VNET/VIMAGE
#ip4 = inherit;
interface = igc0bridge;
vnet;
vnet.interface = "${epair}b";
# common NETWORK config
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

.include "/etc/jail.conf.d/*.conf";
```

- `/etc/jail.conf.d/dev.conf` 独立配置部分:

```bash
dev {
  # NETWORKS/INTERFACES
  $id = "253";
  $ip = "192.168.7.${id}/24";
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

### 脚本辅助配置jail

- 写了一个简单的脚本帮助创建配置文件:

```bash title="jail_zfs.sh"
export jail_zfs="zdata/jails"
export bsd_ver="14.2"

jail_name=$1
id=$2

zfs clone $jail_zfs/templates/$bsd_ver-RELEASE-skeleton@base $jail_zfs/containers/$jail_name
mkdir -p /$jail_zfs/$jail_name-nullfs-base

cat << EOF > /$jail_zfs/$jail_name-nullfs-base.fstab
/$jail_zfs/templates/14.2-RELEASE-base  /$jail_zfs/$jail_name-nullfs-base/         nullfs  ro  0 0
/$jail_zfs/containers/$jail_name               /$jail_zfs/$jail_name-nullfs-base/skeleton nullfs  rw  0 0
EOF

cat << 'EOF' > /etc/jail.conf.d/$jail_name.conf
JAIL_NAME {
  $id = "ID";
  $ip = "192.168.7.${id}/24";
}
EOF

sed -i '' "s@JAIL_NAME@$jail_name@" /etc/jail.conf.d/$jail_name.conf
sed -i '' "s@ID@$id@" /etc/jail.conf.d/$jail_name.conf
```

通过执行

```bash
./jail_zfs.sh pg-1 111
```

就可以创建一个使用 `192.168.7.111` 为IP的名为 `pg-1` 的Jail
