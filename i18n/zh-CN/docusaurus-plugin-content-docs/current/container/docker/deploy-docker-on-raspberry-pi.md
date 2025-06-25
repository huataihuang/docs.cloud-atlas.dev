---
sidebar_position: 2
---

# 在树莓派上部署Docker

使用树莓派4和5运行Docker，为Kubernetes (k0s 或 k3s)做准备

## 安装Docker

:::tip
树莓派操作系统Raspberry Pi OS(64位)是基于Debian的定制版本，所以安装Docker的方法和Debian是一样的。

我使用Docker官方提供的Docker Engine for Debian 兼容 x86_64 (amd64), armhf, arm64, 以及 ppc64le (ppc64el) 架构
:::

### 卸载旧版本

安装Docker官方的Docker Engine之前，务必卸载系统上任何冲突版本:

- docker.io
- docker-compose
- docker-doc
- podman-docker

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
```
### 安装

- 设置 Debian(Raspberry Pi OS) 的Docker apt仓库:

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

- 对于墙内用户可能需要配置 [curl代理](../../os/proxy) :
