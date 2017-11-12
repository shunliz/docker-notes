# Docker 网络

---

1. ## overlay网络![](/assets/docker-overlay.png)
2. ## macvlan

![](/assets/docker-macvlan.png)

## flannel

flannel 是 CoreOS 开发的容器网络解决方案。flannel 为每个 host 分配一个 subnet，容器从此 subnet 中分配 IP，这些 IP 可以在 host 间路由，容器间无需 NAT 和 port mapping 就可以跨主机通信。

每个 subnet 都是从一个更大的 IP 池中划分的，flannel 会在每个主机上运行一个叫 flanneld 的 agent，其职责就是从池子中分配 subnet。为了在各个主机间共享信息，flannel 用 etcd（与 consul 类似的 key-value 分布式数据库）存放网络配置、已分配的 subnet、host 的 IP 等信息。

数据包如何在主机间转发是由 backend 实现的。flannel 提供了多种 backend，最常用的有 vxlan 和 host-gw，我们将在本章讨论这两种 backend。其他 backend 请参考[https://github.com/coreos/flannel。](https://github.com/coreos/flannel。)

![](/assets/docker-flannel.png)**flannel 网络隔离**

flannel 为每个主机分配了独立的 subnet，但 flannel.1 将这些 subnet 连接起来了，相互之间可以路由。本质上，flannel 将各主机上相互独立的 docker0 容器网络组成了一个互通的大网络，实现了容器跨主机通信。flannel 没有提供隔离。

#### **flannel 与外网连通性**

因为 flannel 网络利用的是默认的 bridge 网络，所以容器与外网的连通方式与 bridge 网络一样，即：

1. 容器通过 docker0 NAT 访问外网

2. 通过主机端口映射，外网可以访问容器



