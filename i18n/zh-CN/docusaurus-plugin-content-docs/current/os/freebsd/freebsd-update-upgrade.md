---
sidebar_position: 2
---

# FreeBSD 更新和升级

保持系统更新!

## FreeBSD 系统更新

FreeBSD提供了两种更新系统方式:

- freebsd-update 命令用于更新核心系统
- 包管理器或 `ports系统` ( `ports Collection` ) 用于更新第三方软件

## FreeBSD 系统升级

:::tip
恰逢6月10日FreeBSD发布了最新的 RELEASE-14.3 ，所以我准备借此实践将原本 `RELEASE-14.2` 到升级 `RELEASE-14.3`
:::

- 在执行 **系统升级** 前，先完成前文所述的 **系统更新** ，确保 `当前RELEASE` 是最新:

```bash
freebsd-update fetch install
```

- `RELEASE-14.2` 到升级 `RELEASE-14.3` :

```bash
freebsd-update -r 14.3-RELEASE upgrade
```

此时输出信息:

```
src component not installed, skipped
Looking up update.FreeBSD.org mirrors... 3 mirrors found.
Fetching metadata signature for 14.2-RELEASE from update1.freebsd.org... done.
Fetching metadata index... done.
Fetching 1 metadata files... ^[done.
Inspecting system... done.

The following components of FreeBSD seem to be installed:
kernel/generic world/base

The following components of FreeBSD do not seem to be installed:
kernel/generic-dbg world/base-dbg world/lib32 world/lib32-dbg

// highlight-start
Does this look reasonable (y/n)? y
// highlight-end

Fetching metadata signature for 14.3-RELEASE from update1.freebsd.org... done.
Fetching metadata index... done.
Fetching 1 metadata patches. done.
Applying metadata patches... done.
Fetching 1 metadata files... done.
Inspecting system... done.
Fetching files from 14.2-RELEASE for merging... done.
Preparing to download files... done.
Fetching 3160 patches. done.
Applying patches... done.
Fetching 3316 files...
这里显示下载文件的进度，直到所有3316个文件下载完毕
```
