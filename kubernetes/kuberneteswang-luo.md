**1、容器间通信：**

同一个[Pod](https://www.kubernetes.org.cn/tags/pod)的容器共享同一个网络命名空间，它们之间的访问可以用localhost地址 + 容器端口就可以访问。

![](/assets/kubnet1.png)

**2、同一Node中Pod间通信：**

同一Node中Pod的默认路由都是docker0的地址，由于它们关联在同一个docker0网桥上，地址网段相同，所有它们之间应当是能直接通信的

![](/assets/kubnet2.png)

**3、不同Node中Pod间通信：**

不同Node中Pod间通信要满足2个条件： Pod的IP不能冲突； 将Pod的IP和所在的Node的IP关联起来，通过这个关联让Pod可以互相访问。

![](/assets/kubnet3.png)

**4、Service介绍：**

Service是一组Pod的服务抽象，相当于一组Pod的LB，负责将请求分发给对应的

Pod；Service会为这个LB提供一个IP，一般称为ClusterIP。

![](/assets/kubservice1.png)

![](/assets/kubservice2.png)

**5、Kube-proxy介绍：**

Kube-proxy是一个简单的网络代理和负载均衡器，它的作用主要是负责Service的实现，具体来说，就是实现了内部从Pod到Service和外部的从NodePort向Service的访问。

**实现方式：**

* userspace是在用户空间，通过kuber-proxy实现LB的代理服务，这个是kube-proxy的最初的版本，较为稳定，但是效率也自然不太高。

* 
* iptables是纯采用iptables来实现LB，是目前kube-proxy默认的方式。

**下面是iptables模式下Kube-proxy的实现方式：**

![](/assets/kubproxy1.png)

在这种模式下，kube-proxy监视Kubernetes主服务器添加和删除服务和端点对象。对于每个服务，它安装iptables规则，捕获到服务的clusterIP（虚拟）和端口的流量，并将流量重定向到服务的后端集合之一。对于每个Endpoints对象，它安装选择后端Pod的iptables规则。

* 默认情况下，后端的选择是随机的。可以通过将service.spec.sessionAffinity设置为“ClientIP”（默认为“无”）来选择基于客户端IP的会话关联。

* 与用户空间代理一样，最终结果是绑定到服务的IP:端口的任何流量被代理到适当的后端，而客户端不知道关于Kubernetes或服务或Pod的任何信息。这应该比用户空间代理更快，更可靠。然而，与用户空间代理不同，如果最初选择的Pod不响应，则iptables代理不能自动重试另一个Pod，因此它取决于具有工作准备就绪探测。

**6、Kube-dns介绍**

Kube-dns用来为kubernetes service分配子域名，在集群中可以通过名称访问service；通常kube-dns会为service赋予一个名为“service名称.namespace.svc.cluster.local”的A记录，用来解析service的clusterip。

**Kube-dns组件：**

* 在Kubernetes v1.4版本之前由“Kube2sky、Etcd、Skydns、Exechealthz”四个组件组成。

* 在Kubernetes v1.4版本及之后由“Kubedns、dnsmasq、exechealthz”三个组件组成。

![](/assets/kubdns1.png)

** Kubedns**

* 接入SkyDNS，为dnsmasq提供查询服务。
* 替换etcd容器，使用树形结构在内存中保存DNS记录。
* 通过K8S API监视Service资源变化并更新DNS记录。
* 服务10053端口。

**Dnsmasq**

* Dnsmasq是一款小巧的DNS配置工具。

* 在kube-dns插件中的作用是：

* 通过kubedns容器获取DNS规则，在集群中提供DNS查询服务

* 提供DNS缓存，提高查询性能

* 降低kubedns容器的压力、提高稳定性

* Dockerfile在GitHub上Kubernetes组织的contrib仓库中，位于dnsmasq目录下。

* 在kube-dns插件的编排文件中可以看到，dnsmasq通过参数–server=127.0.0.1:10053指定upstream为kubedns。

**Exechealthz**

* 在kube-dns插件中提供健康检查功能。
* 源码同样在contrib仓库中，位于exec-healthz目录下。
* 新版中会对两个容器都进行健康检查，更加完善。

## **四、Kubernetes网络开源组件**

**隧道方案（ Overlay Networking ）**

隧道方案在IaaS层的网络中应用也比较多，大家共识是随着节点规模的增长复杂度会提升，而且出了网络问题跟踪起来比较麻烦，大规模集群情况下这是需要考虑的一个点。

* **Weave：**
  UDP广播，本机建立新的BR，通过PCAP互通
* **Open vSwitch（OVS）：**
  基于VxLan和GRE协议，但是性能方面损失比较严重
* **Flannel：**
  UDP广播，VxLan
* **Racher：**
  IPsec

**路由方案**

路由方案一般是从3层或者2层实现隔离和跨主机容器互通的，出了问题也很容易排查。

* **Calico：**
  基于BGP协议的路由方案，支持很细致的ACL控制，对混合云亲和度比较高。
* **Macvlan：**
  从逻辑和Kernel层来看隔离性和性能最优的方案，基于二层隔离，所以需要二层路由器支持，大多数云服务商不支持，所以混合云上比较难以实现。

**4、Flannel容器网络：**

Flannel之所以可以搭建kubernets依赖的底层网络，是因为它可以实现以下两点：

* 它给每个node上的docker容器分配相互不想冲突的IP地址；
* 它能给这些IP地址之间建立一个覆盖网络，同过覆盖网络，将数据包原封不动的传递到目标容器内。

**Flannel介绍**

* Flannel是CoreOS团队针对Kubernetes设计的一个网络规划服务，简单来说，它的功能是让集群中的不同节点主机创建的Docker容器都具有全集群唯一的虚拟IP地址。
* 在默认的Docker配置中，每个节点上的Docker服务会分别负责所在节点容器的IP分配。这样导致的一个问题是，不同节点上容器可能获得相同的内外IP地址。并使这些容器之间能够之间通过IP地址相互找到，也就是相互ping通。
* Flannel的设计目的就是为集群中的所有节点重新规划IP地址的使用规则，从而使得不同节点上的容器能够获得“同属一个内网”且”不重复的”IP地址，并让属于不同节点上的容器能够直接通过内网IP通信。
* Flannel实质上是一种“覆盖网络\(overlaynetwork\)”，也就是将TCP数据包装在另一种网络包里面进行路由转发和通信，目前已经支持udp、vxlan、host-gw、aws-vpc、gce和alloc路由等数据转发方式，默认的节点间数据通信方式是UDP转发。

![](/assets/kubflannel1.png)

**5、Calico容器网络：**

**Calico介绍**

* Calico是一个纯3层的数据中心网络方案，而且无缝集成像OpenStack这种IaaS云架构，能够提供可控的VM、容器、裸机之间的IP通信。Calico不使用重叠网络比如flannel和libnetwork重叠网络驱动，它是一个纯三层的方法，使用虚拟路由代替虚拟交换，每一台虚拟路由通过BGP协议传播可达信息（路由）到剩余数据中心。
* Calico在每一个计算节点利用Linux Kernel实现了一个高效的vRouter来负责数据转发，而每个vRouter通过BGP协议负责把自己上运行的workload的路由信息像整个Calico网络内传播——小规模部署可以直接互联，大规模下可通过指定的BGP route reflector来完成。
* Calico节点组网可以直接利用数据中心的网络结构（无论是L2或者L3），不需要额外的NAT，隧道或者Overlay Network。
* Calico基于iptables还提供了丰富而灵活的网络Policy，保证通过各个节点上的ACLs来提供Workload的多租户隔离、安全组以及其他可达性限制等功能。

![](/assets/kubcalico1.png)

**性能对比总结：**

CalicoBGP 方案最好，不能用 BGP 也可以考虑 Calico ipip tunnel 方案；如果是 Coreos 系又能开 udp offload，flannel 是不错的选择；Docker 原生Overlay还有很多需要改进的地方。

![](/assets/kubnetcmp.png)

