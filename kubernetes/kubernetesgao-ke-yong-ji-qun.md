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
    "${NODE_IP}"
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

```
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
ExecStart=/root/local/bin/etcd \\
  --name=${NODE_NAME} \\
  --cert-file=/etc/etcd/ssl/etcd.pem \\
  --key-file=/etc/etcd/ssl/etcd-key.pem \\
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \\
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --initial-advertise-peer-urls=https://${NODE_IP}:2380 \\
  --listen-peer-urls=https://${NODE_IP}:2380 \\
  --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://${NODE_IP}:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

其中NODE\_IP为当前操作ip地址， ETCD\_NODES= master1=https://192.168.2.210:2380,master2=https://192.168.2.211:2380,master3=https://192.168.2.212:2380

```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

# 测试验证etcd

```
etcdctl set testdir/testkey0  0
etcdctl get testdir/testkey0
```

# 

# 安装kubernetes

参考：[https://wiki.shileizcc.com/display/KUB/Kubernetes+HA+Cluster+Build](https://wiki.shileizcc.com/display/KUB/Kubernetes+HA+Cluster+Build)

所有节点执行：

[https://dl.k8s.io/v1.8.1/kubernetes-client-linux-amd64.tar.gz](https://dl.k8s.io/v1.8.1/kubernetes-client-linux-amd64.tar.gz)

[https://dl.k8s.io/v1.8.1/kubernetes-server-linux-amd64.tar.gz](https://dl.k8s.io/v1.8.1/kubernetes-server-linux-amd64.tar.gz)

[https://dl.k8s.io/v1.8.1/kubernetes-node-linux-amd64.tar.gz](https://dl.k8s.io/v1.8.1/kubernetes-node-linux-amd64.tar.gz)

```
tar zxf kubernetes-server-linux-amd64.tar.gz && cp kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/
```

#####  {#KubernetesHAClusterBuild-kube-apiserver部署}

##### kube-apiserver 部署 {#KubernetesHAClusterBuild-kube-apiserver部署}

`mkdir -p /var/log/kubernetes`

`exportINTERNAL_IP=192.168。2.210`

