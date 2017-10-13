**1、容器间通信：**

同一个[Pod](https://www.kubernetes.org.cn/tags/pod)的容器共享同一个网络命名空间，它们之间的访问可以用localhost地址 + 容器端口就可以访问。

![](/assets/kubnet1.png)

**2、同一Node中Pod间通信：**

同一Node中Pod的默认路由都是docker0的地址，由于它们关联在同一个docker0网桥上，地址网段相同，所有它们之间应当是能直接通信的

![](/assets/kubnet2.png)

