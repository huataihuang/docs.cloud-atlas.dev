---
sidebar_position: 2
---

# FreeBSD ZFS

ZFS是最好的文件系统

## 什么是 ZFS

## `zdata` 数据存储

在 [组装服务器](../../hardware/assembly-machine) 中我使用了 **4块** NVMe存储，为了能够充分使用空间，采用了 ``stripe`` 模式。

:::tip
**4块** NVMe存储中的第一块 `nda0` 前 `256GB` 空间安装 FreeBSD 操作系统，所以我对齐另外3块存储去除前 `256GB` 保留他用(预留给glusterfs和数据备份)，只使用后 `1.6T` 空间来组成一个新的ZFS存储池 `zdata`
:::

### 使用的磁盘列表

- 使用 `geom` 检查磁盘列表

```bash
geom disk list
```

可以看到主机安装了4块NVMe磁盘，设备名是 `ndaX` :

```js
// highlight-start
Geom name: nda0
// highlight-end
Providers:
1. Name: nda0
   Mediasize: 2000398934016 (1.8T)
   Sectorsize: 512
   Mode: r0w0e0
   descr: KIOXIA-EXCERIA G2 SSD
   lunid: 00000000000000008ce38e03009ae4ee
// highlight-start
   ident: 54UA4062K7AS
// highlight-end
   rotationrate: 0
   fwsectors: 0
   fwheads: 0

// highlight-start
Geom name: nda1
// highlight-end
Providers:
1. Name: nda1
   Mediasize: 2000398934016 (1.8T)
   Sectorsize: 512
   Mode: r0w0e0
   descr: KIOXIA-EXCERIA G2 SSD
   lunid: 00000000000000008ce38e0300a242e3
// highlight-start
   ident: X4GA30KZKQJP
// highlight-end
   rotationrate: 0
   fwsectors: 0
   fwheads: 0

// highlight-start
Geom name: nda2
// highlight-end
Providers:
1. Name: nda2
   Mediasize: 2000398934016 (1.8T)
   Sectorsize: 512
   Mode: r0w0e0
   descr: KIOXIA-EXCERIA G2 SSD
   lunid: 00000000000000008ce38e03009ae512
// highlight-start
   ident: 54UA4072K7AS
// highlight-end
   rotationrate: 0
   fwsectors: 0
   fwheads: 0

// highlight-start
Geom name: nda3
// highlight-end
Providers:
1. Name: nda3
   Mediasize: 2000398934016 (1.8T)
   Sectorsize: 512
   Mode: r1w1e3
   descr: KIOXIA-EXCERIA G2 SSD
   lunid: 00000000000000008ce38e030091c40e
// highlight-start
   ident: Y39B70RTK7AS
// highlight-end
   rotationrate: 0
   fwsectors: 0
   fwheads: 0
```

### gpart 划分分区

- 已经安装操作系统的磁盘剩余空间划分一个分区

第3块 `diskid/DISK-Y39B70RTK7AS` 分区如下 ( 这块存储目前已经安装了操作系统，即设备 `/dev/diskid/DISK-Y39B70RTK7AS` ，这里没有使用 `/dev/nda3` 而是使用 `diskid` 来标示设备)

```bash
gpart show diskid/DISK-Y39B70RTK7AS
```

可以看到这块磁盘已经安装了FreeBSD操作系统(占用了256GB空间)，还剩下 `1.6TB` 空间未使用

```
=>        40  3907029088  diskid/DISK-Y39B70RTK7AS  GPT  (1.8T)
          40      532480                         1  efi  (260M)
      532520        2008                            - free -  (1.0M)
      534528     4194304                         2  freebsd-swap  (2.0G)
     4728832   536870912                         3  freebsd-zfs  (256G)
// highlight-start
   541599744  3365429384                            - free -  (1.6T)
// highlight-end
```

执行以下命令添加一个分区，这里没有指定 `-s` 参数(容量大小)，所以会把所有剩余空间用完

```js
gpart add -t freebsd-zfs -a 1M diskid/DISK-Y39B70RTK7AS
```

完成后可以看到增加了一个分区

```
=>        40  3907029088  diskid/DISK-Y39B70RTK7AS  GPT  (1.8T)
          40      532480                         1  efi  (260M)
      532520        2008                            - free -  (1.0M)
      534528     4194304                         2  freebsd-swap  (2.0G)
     4728832   536870912                         3  freebsd-zfs  (256G)
// highlight-start
   541599744  3365429248                         4  freebsd-zfs  (1.6T)
// highlight-end
  3907028992         136                            - free -  (68K)
```

- 另外3块完全空白的磁盘划分分区: 分区1 `256GB` ，分区2 `1.6TB` ，其中分区2将用于组合成ZFS

```bash
# 划分256G分区1和剩余空间分区2
gpart add -t freebsd-ufs -a 1M -s 256G diskid/DISK-54UA4062K7AS
gpart add -t freebsd-zfs -a 1M diskid/DISK-54UA4062K7AS

gpart add -t freebsd-ufs -a 1M -s 256G diskid/DISK-X4GA30KZKQJP
gpart add -t freebsd-zfs -a 1M diskid/DISK-X4GA30KZKQJP

gpart add -t freebsd-ufs -a 1M -s 256G diskid/DISK-54UA4072K7AS
gpart add -t freebsd-zfs -a 1M diskid/DISK-54UA4072K7AS
```

- 所有磁盘划分完成之后，最终 `gpart show` 显示如下

```
=>        34  3907029101  diskid/DISK-54UA4062K7AS  GPT  (1.8T)
          34        2014                            - free -  (1.0M)
        2048   536870912                         1  freebsd-ufs  (256G)
// highlight-start
   536872960  3370156032                         2  freebsd-zfs  (1.6T)
// highlight-end
  3907028992         143                            - free -  (72K)

=>        34  3907029101  diskid/DISK-X4GA30KZKQJP  GPT  (1.8T)
          34        2014                            - free -  (1.0M)
        2048   536870912                         1  freebsd-ufs  (256G)
// highlight-start
   536872960  3370156032                         2  freebsd-zfs  (1.6T)
// highlight-end
  3907028992         143                            - free -  (72K)

=>        34  3907029101  diskid/DISK-54UA4072K7AS  GPT  (1.8T)
          34        2014                            - free -  (1.0M)
        2048   536870912                         1  freebsd-ufs  (256G)
// highlight-start
   536872960  3370156032                         2  freebsd-zfs  (1.6T)
// highlight-end
  3907028992         143                            - free -  (72K)

=>        40  3907029088  diskid/DISK-Y39B70RTK7AS  GPT  (1.8T)
          40      532480                         1  efi  (260M)
      532520        2008                            - free -  (1.0M)
      534528     4194304                         2  freebsd-swap  (2.0G)
     4728832   536870912                         3  freebsd-zfs  (256G)
// highlight-start
   541599744  3365429248                         4  freebsd-zfs  (1.6T)
// highlight-end
  3907028992         136                            - free -  (68K)
```

### 创建 `stripe` 模式 ZFS pool `zdata`

:::warning
- 构建的是 `stripe` 模式，是为了追求实验环境容量最大化， **生产环境不可使用!!!**
- 生产环境请使用 `RAIDZ-5`
:::

- 使用上述 `gpart` 划分好的分区2(操作系统所在盘是分区4)来构建一个容量最大化的 `stripe` ZFS 存储池:

```bash
zpool create -f -o ashift=12 zdata /dev/diskid/DISK-Y39B70RTK7ASp4 \
/dev/diskid/DISK-54UA4062K7ASp2 \
/dev/diskid/DISK-X4GA30KZKQJPp2 \
/dev/diskid/DISK-54UA4072K7ASp2
```

:::tip
- `-o ashift=12` 的 `ashift` 属性设置为 `12` ，以对应 4KiB (4096字节)块大小。通常对于HDD和SSD，能够获得较好的性能和兼容性。底层是 512字节 一个扇区，所以 `2^12` = **4096** 就能够对齐和整块读写磁盘。如果没有指定这个 ashift 参数，ZFS会自动检测 ashift ，如果检测失败就会默认使用 `ashift=9` ，这会导致性能损失。这个 ashift 参数一旦设置，不能修改
- 这里使用了 `diskid` 来标记磁盘，以避免搞错磁盘
:::

- 创建zfs卷集

```bash
# 存储文档的卷集，其他卷集主要在Jails中使用
zfs create zdata/docs
```
