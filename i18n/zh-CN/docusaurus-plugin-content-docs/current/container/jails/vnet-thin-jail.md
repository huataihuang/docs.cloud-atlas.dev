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
