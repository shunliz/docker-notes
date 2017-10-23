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

三台机器同时安装etcd

```
# yum install etcd -y
```

yum安装的etcd默认配置文件在/etc/etcd/etcd.conf，以下将三个节点上的配置贴出来，请注意不同点（未贴出的，则表明不需要更改）

master1

```
# [member]
ETCD_NAME=master1
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""
#ETCD_SNAPSHOT_COUNT="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379, http://0.0.0.0:4001"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#
#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://master1:2380"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="master1=http://master1:2380,master2=http://master2:2380,master3=http://master3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://master1:2379,http://master1:4001"
```

master2

```
# [member]
ETCD_NAME=master2
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""
#ETCD_SNAPSHOT_COUNT="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#
#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://master2:2380"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="master1=http://master1:2380,master2=http://master2:2380,master3=http://master3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://master2:2379,http://master2:4001"
```

master3

```
# [member]
ETCD_NAME=master3
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""
#ETCD_SNAPSHOT_COUNT="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#
#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://master3:2380"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="master1=http://master1:2380,master2=http://master2:2380,master3=http://master3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://master3:2379,http://master3:4001"
```

改好配置之后，在各个节点上开启etcd服务\(需要关闭firewalld，否则集群建立不起来: systemctl stop firewalld.service\)：

```
# systemctl restart etcd
```

测试验证etcd

```
etcdctl set testdir/testkey0  0
etcdctl get testdir/testkey0
```

# 安装kubernetes

参考：https://wiki.shileizcc.com/display/KUB/Kubernetes+HA+Cluster+Build

所有节点执行：

https://dl.k8s.io/v1.8.1/kubernetes-client-linux-amd64.tar.gz

https://dl.k8s.io/v1.8.1/kubernetes-server-linux-amd64.tar.gz

https://dl.k8s.io/v1.8.1/kubernetes-node-linux-amd64.tar.gz

```
tar zxf kubernetes-server-linux-amd64.tar.gz && cp kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/
```



