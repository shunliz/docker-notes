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

** Kubedns**

* 接入SkyDNS，为dnsmasq提供查询服务。
* 替换etcd容器，使用树形结构在内存中保存DNS记录。
* 通过K8S API监视Service资源变化并更新DNS记录。
* 服务10053端口。

**Dnsmasq**

* Dnsmasq是一款小巧的DNS配置工具。

* 在kube-dns插件中的作用是：

1. 通过kubedns容器获取DNS规则，在集群中提供DNS查询服务
2. 提供DNS缓存，提高查询性能
3. 降低kubedns容器的压力、提高稳定性

* Dockerfile在GitHub上Kubernetes组织的contrib仓库中，位于dnsmasq目录下。

* 在kube-dns插件的编排文件中可以看到，dnsmasq通过参数–server=127.0.0.1:10053指定upstream为kubedns。

**Exechealthz**

* 在kube-dns插件中提供健康检查功能。
* 源码同样在contrib仓库中，位于exec-healthz目录下。
* 新版中会对两个容器都进行健康检查，更加完善。



