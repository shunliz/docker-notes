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

