---
sidebar_position: 1
---

# FreeBSD 蓝图

最适合作为服务器的操作系统，和Linux不一样的体验

## FreeBSD部署规划

我主要在3个方面使用FreeBSD:

- 阿里云租用虚拟机: 作为对外网关，使用nginx反向代理，将外部WEB访问转发到位于家里的模拟云计算系统
- 主要存储和底层服务器: 使用了一台组装台式机，结合Jail+bhyve来实现运行基础服务
  - Ceph集群模拟(为OpenStack和Kubernetes提供存储服务，尽可能使用精简操作系统，而将所有数据和镜像都通过网络集中存储到Ceph)
  - PostgreSQL集群模拟(为GitLab提供数据存储)
  - etcd集群(为多个Kubernetes模拟集群提供持久化存储)
- 笔记本电脑日常桌面: 不折腾，只使用最基本xfce桌面，因为我的主要开发运维工作都集中在服务器端
