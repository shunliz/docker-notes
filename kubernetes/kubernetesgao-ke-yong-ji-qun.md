etcd集群安装

三台虚拟机，系统环境均为Centos7，对应节点名称及IP地址如下：  
　　　　master1:192.168.2.210  
　　　　master22:192.168.2,211  
　　　　master3:192.168.2.212

首先将这个信息添加到三台主机的hosts文件中，编辑/etc/hosts，填入以下信息：

```
192.168.2.210 master1
192.168.2.211 master2
192.168.2.212 master3
192.168.2.213 node1
192.168.2.214 node2
```

三台 机器创建必须目录 mkdir -p /root/local/bin, mkdir -p /etc/kubernetes/ssl/, mkdir -p /etc/etcd/ssl/

三台机器安装cfssl

```
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ chmod +x cfssl_linux-amd64
$ sudo mv cfssl_linux-amd64 /root/local/bin/cfssl

$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ chmod +x cfssljson_linux-amd64
$ sudo mv cfssljson_linux-amd64 /root/local/bin/cfssljson

$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
$ chmod +x cfssl-certinfo_linux-amd64
$ sudo mv cfssl-certinfo_linux-amd64 /root/local/bin/cfssl-certinfo

$ export PATH=/root/local/bin:$PATH
```

其中一台机器生成根CA信息

```
$ cat ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "8760h"
      }
    }
  }
}
$ cat ca-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
$ ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
$

$ sudo mkdir -p /etc/kubernetes/ssl
$ sudo cp ca* /etc/kubernetes/ssl
```

然后拷贝到其他几个机器的对应目录。

##### 三台机器配置etcd TLS {#KubernetesHAClusterBuild-kube-apiserver部署}

其中NODE\_IP需要修改成当前操作机器IP

```
$ cat > etcd-csr.json <<EOF
{
    "CN": "etcd",
    "hosts": [
      "127.0.0.1",
      "192.168.2.210",
      "192.168.2.211",
      "192.168.2.212",
      "master1",
      "master2",
      "master3"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

## 创建 etcd 的 systemd unit 文件

```bash
[root@master3 ~]# cat /usr/lib/systemd/system/etcd.service 
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/root/local/bin/etcd --name master1 --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem --peer-cert-file=/etc/etcd/ssl/etcd.pem --peer-key-file=/etc/etcd/ssl/etcd-key.pem --trusted-ca-file=/etc/kubernetes/ssl/ca.pem --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem  --initial-advertise-peer-urls=https://192.168.2.210:2380  --listen-peer-urls=https://192.168.2.210:2380 --listen-client-urls=https://192.168.2.210:2379,https://127.0.0.1:2379 --advertise-client-urls=https://192.168.2.210:2379 --initial-cluster-token=etcd-cluster-0 --initial-cluster=master1=https://192.168.2.210:2380,master2=https://192.168.2.211:2380,master3=https://192.168.2.212:2380  --initial-cluster-state=new --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

其中NODE\_IP为当前操作ip地址， ETCD\_NODES= master1=[https://192.168.2.210:2380,master2=https://192.168.2.211:2380,master3=https://192.168.2.212:2380](https://192.168.2.210:2380,master2=https://192.168.2.211:2380,master3=https://192.168.2.212:2380)

```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

# 测试验证etcd

```
etcdctl \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --endpoints=https://master1:2379,https://master2:2379,https://master3:2379 \
  cluster-health

etcdctl set testdir/testkey0  0
etcdctl get testdir/testkey0
```



# Pacemaker Corosync Kubernetes VIP {#Pacemaker-PacemakerCorosyncKubernetesVIP}

```
yum install -y pcs pacemaker corosync fence-agents-all resource-agent
```



# 安装kubernetes

参考：[https://wiki.shileizcc.com/display/KUB/Kubernetes+HA+Cluster+Build](https://wiki.shileizcc.com/display/KUB/Kubernetes+HA+Cluster+Build)

[https://github.com/opsnull/follow-me-install-kubernetes-cluster](https://github.com/opsnull/follow-me-install-kubernetes-cluster)

所有节点执行：

wget [https://dl.k8s.io/v1.8.1/kubernetes-client-linux-amd64.tar.gz](https://dl.k8s.io/v1.8.1/kubernetes-client-linux-amd64.tar.gz)

wget [https://dl.k8s.io/v1.8.1/kubernetes-server-linux-amd64.tar.gz](https://dl.k8s.io/v1.8.1/kubernetes-server-linux-amd64.tar.gz)

wget [https://dl.k8s.io/v1.8.1/kubernetes-node-linux-amd64.tar.gz](https://dl.k8s.io/v1.8.1/kubernetes-node-linux-amd64.tar.gz)

```
tar zxf kubernetes-server-linux-amd64.tar.gz && cp kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/
```

##### 创建 kube-apiserver 证书和私钥 {#KubernetesHAClusterBuild-创建kube-apiserver证书和私钥}

创建 kube-apiserver 证书签名请求配置

```
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "192.166.1.12",
      "192.166.1.2",
      "192.166.1.13",
      "10.96.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "cloudnative"
        }
    ]
}
```

## 

## 

## 

## 

## 创建 admin 证书

kubectl 与 kube-apiserver 的安全端口通信，需要为安全通信提供 TLS 证书和秘钥。

创建 admin 证书签名请求

```
$ cat admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

生成 admin 证书和私钥：

```
$ cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin
$ ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem
$ sudo mv admin*.pem /etc/kubernetes/ssl/
$ rm admin.csr admin-csr.json
$
```

```
$ export MASTER_IP=192.168.2.210 # 替换为 kubernetes master 集群任一机器 IP
$ export KUBE_APISERVER="https://${MASTER_IP}:6443"
```

## 创建 kubectl kubeconfig 文件

```
$ # 设置集群参数
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}
$ # 设置客户端认证参数
$ kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem
$ # 设置上下文参数
$ kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin
$ # 设置默认上下文
$ kubectl config use-context kubernetes
```

* `admin.pem`证书 O 字段值为`system:masters`，`kube-apiserver`预定义的 RoleBinding`cluster-admin`将 Group`system:masters`与 Role`cluster-admin`绑定，该 Role 授予了调用`kube-apiserver`相关 API 的权限；

* 生成的 kubeconfig 被保存到`~/.kube/config`文件；

## 创建 TLS 秘钥和证书

etcd 集群启用了双向 TLS 认证，所以需要为 flanneld 指定与 etcd 集群通信的 CA 和秘钥。

创建 flanneld 证书签名请求：

```
$ cat > flanneld-csr.json <<EOF
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

生成 flanneld 证书和私钥：

```
$ cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
$ ls flanneld*
flanneld.csr  flanneld-csr.json  flanneld-key.pem flanneld.pem
$ sudo mkdir -p /etc/flanneld/ssl
$ sudo mv flanneld*.pem /etc/flanneld/ssl
$ rm flanneld.csr  flanneld-csr.json
```

## 安装和配置 flanneld

### 下载 flanneld

```
$ mkdir flannel
$ wget https://github.com/coreos/flannel/releases/download/v0.7.1/flannel-v0.7.1-linux-amd64.tar.gz
$ tar -xzvf flannel-v0.7.1-linux-amd64.tar.gz -C flannel
$ sudo cp flannel/{flanneld,mk-docker-opts.sh} /root/local/bin
```

```
[root@master1 ~]# cat /usr/lib/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/root/local/bin/flanneld 
  -etcd-cafile=/etc/kubernetes/ssl/ca.pem 
  -etcd-certfile=/etc/flanneld/ssl/flanneld.pem 
  -etcd-keyfile=/etc/flanneld/ssl/flanneld-key.pem 
  -etcd-endpoints="https://192.168.2.210:2379,https://192.168.2.211:2379,https://192.168.2.212:2379" 
  -etcd-prefix="/kubernetes/network"
ExecStartPost=/root/local/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service


$ sudo systemctl daemon-reload
$ sudo systemctl enable flanneld
$ sudo systemctl start flanneld
$ systemctl status flanneld
```

##### 部署master {#KubernetesHAClusterBuild-kube-apiserver部署}



