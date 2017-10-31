在正式内容之前，先简单介绍下本人对k8s做的一些功能支持：

* 自实现对接公司LVS的负载均衡控制器\(lb-controoller\)；
* 自实现对接公司Qbus的日志收集控制器\(log-controller\)；
* 修改RollingUpdate逻辑，让滚动更新更加可控，业务升级更加平滑；

以上三个功能都是我们在构建容器化平台过程中,为适配公司现有生态而进行的一系列扩展。

## 我理解的k8s

先来看看k8s的架构图和各个组件

![](https://www.addops.cn/static/upload/20170820/tke5S-9YASUuIpJXoaIuRZkO.png "alt")

### 按角色划分

按照角色划分，k8s和绝大多数分布式系统一样，分为Master和Slave两类。其中Master是整个系统的控制中心，是大脑；而Slave呢是分布在各个节点的工作单元，是神经元。

### 按功能划分

按照功能划分，k8s是由很多松耦合的功能组件构成。

* etcd 组件

  是k8s系统的资源\(数据\)存储的地方，是整个k8s的基石，也是k8s很多功能的设计之源。做为一个高可用强一致性的服务发现存储仓库，etcd本身非常好的提供了数据的持久化存储和服务发现的支持。在云计算时代，如何让服务快速透明地接入到计算集群中，如何让共享配置信息快速被集群中的所有机器发现是迫切要解决的问题，也是最最核心的问题。etcd很好的解决了这个问题。

* kube-apiserver组件‘

  是k8s 的入口，也是整个k8s除了etcd外最最基础、最最核心的组件。它是整个k8s系统和生态的枢纽和消息总线。没有kube-apiserver整个k8s系统将会土崩瓦解，无法运转。

  从功能实现角度来看，kube-apiserver是对etcd的一个包装，通过对etcd的进一步包装，实现两大功能。一是对外提供查询资源对象的接口；二是提供k8s中资源注册与发现的框架。

  第一点大家都能很轻松的理解，这么大的系统，肯定是要提供一个统一的数据查询接口；第二点大家不深入去看k8s各个组件工作机制和源码实现的话，不一定能够体会得到。资源注册与发现框架是整个k8s架构设计的灵魂。所有k8s的组件，除了etcd外，都是围绕k8s的这一机制去扩展实现的。比如下面将要介绍的kube-controller-manager、kube-scheduler、kubelet、kube-proxy，还有我们自己实现的lb-controller和log-controller。

  > 基于etcd实现的资源的注册和发现框架是整个k8s生态的灵魂。

  服务的注册与发现相信大家都很了解了，那资源的注册与发现和它有啥区别吗？

  然而并没有。服务也可以看成是一种资源。服务发现解决的是分布式系统中最常见的问题之一，即在同一个分布式集群中的进程或服务，要如何才能找到对方并建立连接。本质上来说，服务发现就是想要了解集群中是否有进程在监听udp或tcp端口，并且通过名字就可以查找和连接。而对于资源发现而言就是想要了解集群中是否存在某资源，并且通过名字就可以查找和监听资源状态的变化。

  接下来我们看看k8s中其它组件是如何利用k8s的资源注册与发现机制实现与k8s完美对接的。

* kube-controller-manager

  kube-controller-manager 是 k8s 多种资源控制器的集合。像ReplicationController、ReplicaSetController、DeploymentController、ConfigMapController等等。通过这些控制器实现对各种资源的CRUD操作。 这些控制器通过k8s资源注册与发现框架（其中最核心的就是[list/watch机制](https://www.kubernetes.org.cn/174.html)）来注册和发现自己关心的资源状态变化（如：ReplicationController关心RC和POD资源、ReplicaSetController关心RS和POD资源等）。通过感知资源状态的变化，对这些资源进行相应的处理，使得资源状态最终达到规定的状态。

  这里以ReplicaSetController为例进行简单阐述。![](https://www.addops.cn/static/upload/20170820/HaaaRm-98Rk1U6ag3moYhy0l.png "alt")ReplicaSetController是k8s中用来管理POD的控制器，所以主要关心RS本身和POD资源的状态变化。在启动时便会通过apiserver的list/watch机制注册并监听所关心的资源。当用户创建ReplicaSet资源时，ReplicaSetController监听到该对象的创建事件，将通过apiserver获取RS对象，根据对象的资源描述（POD、POD副本数等）去创建相应副本数的POD。当RS中的POD因某种原因挂掉时，ReplicaSetController也会监听到，并重新创建1个新的POD，保证存活POD数不变。

* kube-scheduler

  kube-scheduler 是k8s资源调度组件，负责pod资源落到哪个或哪些node的调度。按照之前对OpenStack调度的理解来看，本以为对于kube-scheduler而言也是接收apiserver的请求然后去做各种资源是否满足需求的处理，但是实际上它并不是直接接收apiserver的请求，而是基于k8s的资源注册与发现框架来发现pod创建事件的，然后将pod入队列等待资源运算处理，最终返回可用的node，并将pod与之绑定。

* kubelet

  k8s是一个分布式的集群管理系统，在每个节点（node）上都要运行一个 agent 对容器进行生命周期的管理，这个 agent 程序就是 kubelet。简单地说，kubelet 的主要功能就是定时从某个地方获取节点上 pod/Container 的期望状态（运行什么容器、运行的副本数量、网络或者存储如何配置等等），并调用对应的容器平台接口达到这个状态。 上面说的定时从某个地方获取就是指的通过k8s资源注册与发现框架从apiserver中获取。

* kube-proxy

  service是一组pod的服务抽象，相当于一组pod的LB，负责将请求分发给对应的pod。service会为这个LB提供一个IP，一般称为cluster IP。kube-proxy的作用主要是负责service的实现，是service的控制器。它主要关心service、endpoint资源的状态变化，kube-proxy通过k8s资源注册与发现框架从apiserver中获取期望的状态，并调用对应的接口使其达到这个状态。

  我们自实现的lb-controller、log-controller

* lb-controller

  截止到1.7.0版本，kubernetes 还没有实现对四层代理的支持，而我司大部分使用的是基于LVS的四层代理，为了实现 kubernetes 与我司 LVS 的对接我们实现了自己的负载均衡控制器lb-controller。lb-controller 主要是利用k8s资源注册与发现框架，从apiserver中获取pod增删事件，根据不同的事件做出不同的操作。对于增pod，会将该pod挂载到 LVS 中与其关联的 vip 下；对于删pod，会将该pod从LVS中的vip下卸载。从而实现动态感知pod变化并更新rs的功能。

* log-controller

  对于kubernetes中应用的日志，我们是通过公司现有的Qbus日志收集系统收集走的。Qbus是我司基于Kafka、logstash和Qconf实现的一套日志收集系统，已被公司绝大部分业务使用。对于要收集日志的机器而言，会在上面部署一个loogstash的agent，负责读取日志并上传给Qbus。logstash的配置是人工参与处理的。对于虚拟机和物理机这种不会频繁变更的计算资源，人工参与配置是可以接受的，但是对于k8s中pod这类生命周期端，频繁变更的资源而言，再通过人工去处理的话，运维人员会疯掉的。为了解决这个问题，我们基于k8s资源注册与发现框架实现了对pod资源的自发现。动态的感知到pod的增删，从而实现自动化的去配置logstash的功能，大大提高了生产效率。

  为了k8s的开发者和用户更加方便扩展k8s的功能，k8s提供了资源注册和发现的客户端库的[client-go](https://github.com/kubernetes/client-go)，通过client-go可以很方便的连接到apiserver并监听资源状态的变化。向下屏蔽了很多细节，开发者和用户只关心实现资源状态变化后的处理逻辑就可以了，[client-go使用介绍](https://www.kubernetes.org.cn/1309.html)。

  ![](https://www.addops.cn/static/upload/20170820/cGrjm_B0ua4kFw7dwOGpJZnN.png "alt")



