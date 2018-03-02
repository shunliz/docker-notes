整个流程梳理：

1，用户通过kubectl创建serviceaccount, 请求会发到controller manager， controller manager启动参数中配置的--service-account-private-key-file 对serviceaccount对应的用户（这里其实是虚拟用户，可能是CN或者一个随机数）进行签名生成serviceaccount对应的secret里边的token。多master的情况下，需要使用server的key，client的key在各个节点不一致。

2，用户创建pod，指定serviceaccount,对应的信息ca.crt, namespace,token这些信息会挂载到pod的/var/run/secrets/kubernetes.io/serviceaccount下。

3， pod中的应用访问k8s api， 这些pod应用通常是使用go-client访问k8s api的， 使用incluster方式初始化client，也就是会去/var/run/secrets/kubernetes.io/serviceaccount寻找token，发送到apiserver，请求访问api。

4， apiserver对token进行验证，由于加密使用的是server.key，是私钥，所以apiserver启动参数--service-account-key-file需要配置成server的公钥，也就是server.crt。

这样整个身份认证过程完成。



[Kubernetes API Server](https://kubernetes.io/docs/admin/kube-apiserver/)是整个[Kubernetes集群](http://tonybai.com/2016/10/18/learn-how-to-install-kubernetes-on-ubuntu/)的核心，我们不仅有从集群外部访问API Server的需求，有时，我们还需要从Pod的内部访问API Server。

然而，在生产环境中，Kubernetes API Server都是“设防”的。在《[Kubernetes集群的安全配置](http://tonybai.com/2016/11/25/the-security-settings-for-kubernetes-cluster/)》一文中，我提到过：Kubernetes通过client cert、static token、basic auth等方法对客户端请求进行[身份验证](https://kubernetes.io/docs/admin/authentication/#authentication-strategies)。对于运行于Pod中的Process而言，有些时候这些方法是适合的，但有些时候，像client cert、static token或basic auth这些信息是不便于暴露给Pod中的Process的。并且通过这些方法通过API Server验证后的请求是具有全部授权的，可以任意操作[Kubernetes cluster](http://tonybai.com/2017/01/24/explore-kubernetes-cluster-installed-by-kubeadm/)，这显然是不能满足安全要求的。为此，Kubernetes更推荐大家使用[service account](https://kubernetes.io/docs/user-guide/service-accounts/)这种方案的。本文就带大家详细说说如何通过service account从一个Pod中访问API Server的。

### 零、试验环境

本文的试验环境是Kubernetes 1.3.7 cluster，双节点，master承载负荷。cluster通过[kube-up.sh](https://kubernetes.io/docs/getting-started-guides/ubuntu/manual/)搭建的，具体的搭建方法见《[一篇文章带你了解Kubernetes安装](http://tonybai.com/2016/10/18/learn-how-to-install-kubernetes-on-ubuntu/)》。

### 一、什么是service account？

什么是service account? 顾名思义，相对于user account（比如：kubectl访问APIServer时用的就是user account），service account就是Pod中的Process用于访问Kubernetes API的account，它为Pod中的Process提供了一种身份标识。相比于user account的全局性权限，service account更适合一些轻量级的task，更聚焦于授权给某些特定Pod中的Process所使用。

service account作为一种resource存在于Kubernetes cluster中，我们可以通过kubectl获取当前cluster中的service acount列表：

```
# kubectl get serviceaccount --all-namespaces
NAMESPACE                    NAME           SECRETS   AGE
default                      default        1         140d
kube-system                  default        1         140d
```

我们查看一下kube-system namespace下名为”default”的service account的详细信息：

```
# kubectl describe serviceaccount/default -n kube-system
Name:        default
Namespace:    kube-system
Labels:        
<
none
>


Image pull secrets:    
<
none
>


Mountable secrets:     default-token-hpni0

Tokens:                default-token-hpni0
```

我们看到service account并不复杂，只是关联了一个secret资源作为token，该token也叫service-account-token，该token才是真正在API Server验证\(authentication\)环节起作用的：

```
# kubectl get secret  -n kube-system
NAME                  TYPE                                  DATA      AGE
default-token-hpni0   kubernetes.io/service-account-token   3         140d

# kubectl get secret default-token-hpni0 -o yaml -n kube-system
apiVersion: v1
data:
  ca.crt: {base64 encoding of ca.crt data}
  namespace: a3ViZS1zeXN0ZW0=
  token: {base64 encoding of bearer token}

kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
    kubernetes.io/service-account.uid: 90ded7ff-9120-11e6-a0a6-00163e1625a9
  creationTimestamp: 2016-10-13T08:39:33Z
  name: default-token-hpni0
  namespace: kube-system
  resourceVersion: "2864"
  selfLink: /api/v1/namespaces/kube-system/secrets/default-token-hpni0
  uid: 90e71909-9120-11e6-a0a6-00163e1625a9
type: kubernetes.io/service-account-token
```

我们看到这个类型为service-account-token的secret资源包含的数据有三部分：ca.crt、namespace和token。

* ca.crt  
  这个是API Server的[CA公钥证书](http://tonybai.com/2015/04/30/go-and-https/)，用于Pod中的Process对API Server的服务端数字证书进行校验时使用的；

* namespace  
  这个就是Secret所在namespace的值的base64编码：\# echo -n “kube-system”\|base64 =&gt; “a3ViZS1zeXN0ZW0=”

* token

这是一段用API Server私钥签发\(sign\)的bearer tokens的base64编码，在API Server authenticating环节，它将派上用场。

### 二、API Server的service account authentication\(身份验证\)

前面说过，service account为Pod中的Process提供了一种身份标识，在Kubernetes的身份校验\(authenticating\)环节，以某个service account提供身份的Pod的用户名为：

```
system:serviceaccount:(NAMESPACE):(SERVICEACCOUNT)
```

以上面那个kube-system namespace下的“default” service account为例，使用它的Pod的username全称为：

```
system:serviceaccount:kube-system:default
```

有了username，那么credentials呢？就是上面提到的service-account-token中的token。在《[Kubernetes集群的安全配置](http://tonybai.com/2016/11/25/the-security-settings-for-kubernetes-cluster/)》一文中我们谈到过，API Server的authenticating环节支持多种身份校验方式：client cert、bearer token、static password auth等，这些方式中有一种方式通过authenticating（Kubernetes API Server会逐个方式尝试），那么身份校验就会通过。一旦API Server发现client发起的request使用的是service account token的方式，API Server就会自动采用signed bearer token方式进行身份校验。而request就会使用携带的service account token参与验证。该token是API Server在创建service account时用API server启动参数：–service-account-key-file的值签署\(sign\)生成的。如果–service-account-key-file未传入任何值，那么将默认使用–tls-private-key-file的值，即API Server的私钥（server.key）。

通过authenticating后，API Server将根据Pod username所在的group：system:serviceaccounts和system:serviceaccounts:\(NAMESPACE\)的权限对其进行[authority](https://kubernetes.io/docs/admin/authorization/) 和[admission control](https://kubernetes.io/docs/admin/admission-controllers/)两个环节的处理。在这两个环节中，cluster管理员可以对service account的权限进行细化设置。

### 三、默认的service account

Kubernetes会为每个cluster中的namespace自动创建一个默认的service account资源，并命名为”default”：

```
# kubectl get serviceaccount --all-namespaces
NAMESPACE                    NAME           SECRETS   AGE
default                      default        1         140d
kube-system                  default        1         140d
```

如果Pod中没有显式指定spec.serviceAccount字段值，那么Kubernetes会将该namespace下的”default” service account自动mount到在这个namespace中创建的Pod里。我们以namespace “default”为例，我们查看一下其中的一个Pod的信息：

```
# kubectl describe pod/index-api-2822468404-4oofr
Name:        index-api-2822468404-4oofr
Namespace:    default
... ...

Containers:
  index-api:
   ... ...
    Volume Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-40z0x (ro)
    Environment Variables:    
<
none
>

... ...
Volumes:
... ...
  default-token-40z0x:
    Type:    Secret (a volume populated by a Secret)
    SecretName:    default-token-40z0x

QoS Class:    BestEffort
Tolerations:    
<
none
>

No events.
```

可以看到，kubernetes将default namespace中的service account “default”的service account token挂载\(mount\)到了Pod中容器的/var/run/secrets/kubernetes.io/serviceaccount路径下。

深入容器内部，查看mount的serviceaccount路径下的结构：

```
# docker exec 3d11ee06e0f8 ls  /var/run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

这三个文件与上面提到的service account的token中的数据是一一对应的。

### 四、default service account doesn’t work

上面提到过，每个Pod都会被自动挂载一个其所在namespace的default service account，该service account用于该Pod中的Process访问API Server时使用。Pod中的Process该怎么用这个service account呢？Kubernetes官方提供了一个[client-go](https://github.com/kubernetes/client-go)项目可以为你演示如何使用service account访问API Server。这里我们就基于client-go项目中的examples/in-cluster/main.go来测试一下是否能成功访问API Server。

先下载client-go源码：

```
# go get k8s.io/client-go

# ls -F
CHANGELOG.md  dynamic/   Godeps/     INSTALL.md   LICENSE   OWNERS  plugin/    rest/     third_party/  transport/  vendor/
discovery/    examples/  informers/  kubernetes/  listers/  pkg/    README.md  testing/  tools/        util/
```

我们改造一下examples/in-cluster/main.go，考虑到panic会导致不便于观察Pod日志，我们将panic改为输出到“标准输出”，并且不return，让Pod周期性的输出相关日志，即便fail：

```
// k8s.io/client-go/examples/in-cluster/main.go
... ...
func main() {
    // creates the in-cluster config
    config, err := rest.InClusterConfig()
    if err != nil {
        fmt.Println(err)
    }
    // creates the clientset
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        fmt.Println(err)
    }
    for {
        pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})
        if err != nil {
            fmt.Println(err)
        } else {
            fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))
        }
        time.Sleep(10 * time.Second)
    }
}
```

基于该main.go的go build默认输出，创建一个简单的Dockerfile：

```
From ubuntu:14.04
MAINTAINER Tony Bai 
<
bigwhite.cn@gmail.com
>


COPY main /root/main
RUN chmod +x /root/main
WORKDIR /root
ENTRYPOINT ["/root/main"]
```

构建一个测试用docker image：

```
# docker build -t k8s/example1:latest .
... ...

# docker images|grep k8s
k8s/example1                                                  latest              ceb3efdb2f91        14 hours ago        264.4 MB
```

创建一份deployment manifest：

```
//main.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: k8s-example1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: k8s-example1
    spec:
      containers:
      - name: k8s-example1
        image: k8s/example1:latest
        imagePullPolicy: IfNotPresent
```

我们来创建该deployment（kubectl create -f main.yaml -n kube-system），观察Pod中的main程序能否成功访问到API Server：

```
# kubectl logs k8s-example1-1569038391-jfxhx
the server has asked for the client to provide credentials (get pods)
the server has asked for the client to provide credentials (get pods)

API Server log(/var/log/upstart/kube-apiserver.log):

E0302 15:45:40.944496   12902 handlers.go:54] Unable to authenticate the request due to an error: crypto/rsa: verification error
E0302 15:45:50.946598   12902 handlers.go:54] Unable to authenticate the request due to an error: crypto/rsa: verification error
E0302 15:46:00.948398   12902 handlers.go:54] Unable to authenticate the request due to an error: crypto/rsa: verification error
```

出错了！kube-system namespace下的”default” service account似乎不好用啊！（注意：这是在kubernetes 1.3.7环境）。

### 五、创建一个新的自用的service account

在kubernetes github issues中，有好多issue是关于”default” service account不好用的问题，给出的解决方法似乎都是创建一个新的service account。

service account的创建非常简单，我们创建一个serviceaccount.yaml：

```
//serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-example1
```

创建该service account：

```
# kubectl create -f serviceaccount.yaml
serviceaccount "k8s-example1" created

# kubectl get serviceaccount
NAME           SECRETS   AGE
default        1         139d
k8s-example1   1         12s
```

修改main.yaml，让Pod显示使用这个新的service account：

```
//main.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: k8s-example1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: k8s-example1
    spec:
      serviceAccount: k8s-example1
      containers:
      - name: k8s-example1
        image: k8s/example1:latest
        imagePullPolicy: IfNotPresent
```

好了，我们重新创建该deployment，查看Pod日志：

```
# kubectl logs k8s-example1-456041623-rqj87
There are 14 pods in the cluster
There are 14 pods in the cluster
... ...
```

我们看到main程序使用新的service account成功通过了API Server的身份验证环节，并获得了cluster的相关信息。

### 六、尾声

在我的另外一个[使用kubeadm安装的k8s 1.5.1环境](http://tonybai.com/2016/12/30/install-kubernetes-on-ubuntu-with-kubeadm/)中，我重复做了上面这个简单测试，不同的是这次我直接使用了default service account。在[k8s 1.5.1](http://tonybai.com/2017/01/24/explore-kubernetes-cluster-installed-by-kubeadm/)下，pod的执行结果是ok的，也就是说通过default serviceaccount，我们的client-go in-cluster example程序可以顺利通过API Server的身份验证，获取到相关的Pods元信息。

### 七、参考资料

* [Kubernetes authentication](https://kubernetes.io/docs/admin/authentication)
* [Service Accounts](https://kubernetes.io/docs/user-guide/service-accounts/)
* [Accessing the cluster](https://kubernetes.io/docs/user-guide/accessing-the-cluster/#accessing-the-api-from-a-pod)
* [Service Accounts Admin](https://kubernetes.io/docs/admin/service-accounts-admin/)

上边对serviceaccount的使用场景，如何工作，客户端的使用方式等讲的比较清楚了。没有提到的就是serviceaccount token是如何验证的。可以看参考资料最后一个，k8s文档讲的也很清楚了。

# 管理服务帐号（Service Accounts）

## 用户账号 vs 服务账号 {#用户账号-vs-服务账号}

Kubernetes 将用户账号和服务账号的概念区分开，主要基于以下几个原因：

* 用户账号是给人使用的。服务账号是给 pod 中运行的进程使用的。
* 用户账号为全局设计的。命名必须在一个集群的所有命名空间中唯一，未来的用户资源不会被设计到命名空间中。 服务账号是在命名空间里的。
* 典型场景中，一个集群的用户账号是从企业数据库的同步来的，在数据库中新用户帐号一般需要特殊权限，并且账号是与复杂的业务流程关联的。 服务账号的创建往往更轻量，允许集群用户为特定的任务创建服务账号（即，权限最小化原则）。
* 对于人和服务的账号，审计要求会是不同的。
* 对于复杂系统而言，配置包可以包含该系统各类组件的服务账号定义， 因为服务账号能被临时创建，并且有命名空间分类的名字，这类配置是便携式的。

## 服务账号自动化 {#服务账号自动化}

服务账号的自动化由三个独立的组件共同配合实现：

* 用户账号准入控制器（Service account admission controller）
* 令牌控制器（Token controller）
* 服务账号控制器（Service account controller）

### 服务账号准入控制器 {#服务账号准入控制器}

对于 pods 的操作是通过一个叫做[准入控制器](https://k8smeetup.github.io/docs/admin/admission-controllers)的插件（plugin）实现的。它是 APIserver 的一部分。 当 pod 被创建或更新时，它会同步更改 pod。当这个插件是活动状态时（大部分版本默认是活动状态），在 pod 被创建或者更改时， 它会做如下操作：

1. 如果 pod 没有配置`ServiceAccount`，它会将`ServiceAccount`设置为`default`。
2. 确保被 pod 关联的`ServiceAccount`是存在的，否则就拒绝请求。
3. 如果 pod 没有包含任何的`ImagePullSecrets`，那么`ServiceAccount`的`ImagePullSecrets`就会被添加到 pod。
4. 它会把`volume`添加给 pod，该 pod 包含有一个用于 API 访问的令牌。
5. 它会把`volumeSource`添加到 pod 的每个容器，挂载到`/var/run/secrets/kubernetes.io/serviceaccount`。

### 令牌控制器 {#令牌控制器}

令牌控制器（TokenController）作为 controller-manager 的一部分运行。它异步运行。它会：

* 监听对于 serviceAccount 的创建动作，并创建对应的 Secret 以允许 API 访问。
* 监听对于 serviceAccount 的删除动作，并删除所有对应的 ServiceAccountToken Secret。
* 监听对于 secret 的添加动作，确保相关联的 ServiceAccount 是存在的，并根据需要为 secret 添加一个令牌。
* 监听对于 secret 的删除动作，并根据需要删除对应 ServiceAccount 的关联。

你必须给令牌控制器传递一个服务帐号的私钥（private key），通过`--service-account-private-key-file`参数完成。传递的私钥将被用来对服务帐号令牌进行签名。 类似的，你必须给 kube-apiserver 传递一个公钥（public key），通过`--service-account-key-file`参数完成。传递的公钥在认证过程中会被用于验证令牌。

#### 创建额外的 API 令牌（API token） {#创建额外的-api-令牌api-token}

控制器的循环运行会确保对于每个服务帐号都存在一个带有 API 令牌的 secret。 如需要为服务帐号创建一个额外的 API 令牌，可以创建一个`ServiceAccountToken`类型的 secret，并添加与服务帐号对应的 annotation 属性，控制器会为它更新令牌：

secret.json:

```
{
    "kind": "Secret",
    "apiVersion": "v1",
    "metadata": {
        "name": "mysecretname",
        "annotations": {
            "kubernetes.io/service-account.name": "myserviceaccount"
        }
    },
    "type": "kubernetes.io/service-account-token"
}
```

```
kubectl create  -f ./secret.json
kubectl describe secret mysecretname
```

#### 删除/作废服务账号令牌 {#删除作废服务账号令牌}

```
kubectl delete secret mysecretname
```

### 服务账号控制器 {#服务账号控制器}

服务帐号控制器在命名空间内管理 ServiceAccount，需要保证名为 “default” 的 ServiceAccount 在每个命名空间中存在。

