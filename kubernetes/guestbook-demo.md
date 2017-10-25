Guestbook定义文件在Kubernetes发布包的examples/guestbook目录下。我选用的是Kubernetes 1.2.7版本下的代码。

Guestbook包含两个部分：  
1）Frontend  
Guestbook的Web前端部分。  
2）Redis  
Guestbook的存储部分。采用主备模式，运行1个Redis Master和两个Redis Slave，Redis Slave从Redis Master同步数据。

Guestbook实现的功能：在Frontend页面提交数据，保存到Redis Master里，然后从Redis Slave读取数据，显示到页面上。

本例子需要提前安装Cluster DNS，通过DNS发现服务。

# 创建Redis-Master Pod {#创建redis-master-pod}

redis-master-deployment.yaml内容如下：

```
apiVersion:
 extensions/v1beta1

kind:
 Deployment

metadata:

  name: redis-master

spec:

  replicas: 1

  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: redis
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379

```

创建Pod

```
# kubectl create -f redis-master-deployment.yaml 

```

# 创建Redis-Master Service {#创建redis-master-service}

redis-master-service.yaml如下：

```
apiVersion:v1

kind:
 Service

metadata:

  name: redis-master
  labels:
    app: redis
    role: master
    tier: backend

spec:

  ports:
  - port: 6379

    targetPort: 6379

  selector:
    app: redis
    role: master
    tier: backend

```

创建Service

```
# kubectl create -f redis-master-service.yaml 

```

# 创建Redis-Slave Pod {#创建redis-slave-pod}

```
apiVersion:
 extensions/v1beta1

kind:
 Deployment

metadata:

  name: redis-slave

spec:

  replicas: 2

  template:
    metadata:
      labels:
        app: redis
        role: slave
        tier: backend
    spec:
      containers:
      - name: slave
        image: redisslave
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - containerPort: 6379

```

创建Pod

```
# kubectl create -f redis-slave-deployment.yaml 

```

# 创建Redis-Slave Service {#创建redis-slave-service}

redis-slave-service.yaml如下：

```
apiVersion:v1

kind:
 Service

metadata:

  name: redis-slave
  labels:
    app: redis
    role: slave
    tier: backend

spec:

  ports:
  - port: 6379

  selector:
    app: redis
    role: slave
    tier: backend

```

创建Service

```
# kubectl create -f redis-slave-service.yaml 

```

# 创建Frontend Pod {#创建frontend-pod}

frontend-deployment.yaml如下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      -
name:
 php-redis

        image: gcr.io/google_samples/gb-frontend:v3
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        -
name:GET_HOSTS_FROM
          value: dns
        ports:
        -
containerPort:80

```

创建Frontend Pod

```
# kubectl create -f frontend-deployment.yaml

```

# 创建Frontend Service {#创建frontend-service}

frontend-service.yaml如下：

```
apiVersion:v1

kind:
 Service

metadata:

  name: frontend
  labels:
    app: guestbook
    tier: frontend

spec:

  type: NodePort
  ports:
  - port: 80

    name: frontend
    nodePort: 30001

  selector:
    app: guestbook
    tier: frontend

```

创建Frontend Service

```
# kubectl create -f frontend-service.yaml

```

# 验证 {#验证}

```
# kubectl get deployment -o wide

```

![](http://img.blog.csdn.net/20170804144336773?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

```
# kubectl get svc -o wide

```

![](http://img.blog.csdn.net/20170804144422356?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")  
~~frontend创建了NodePort，为32009。~~最新版已在frontend-service.yaml中指定NodePort为30001了。避免每次随机建立端口号。

```
# kubectl get pods -o wide

```

![](http://img.blog.csdn.net/20170804144708848?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")  
frontend pod部署在192.168.121.144和192.168.121.145上。  
~~任意打开192.168.121.144:32009和192.168.121.145:32009都可以访问该网页。~~32009改为30001

~~打开192.168.121.144:32009。~~32009改为30001  
输入“node1”。  
![](http://img.blog.csdn.net/20170804145120848?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

~~打开192.168.121.145:32009~~32009改为30001  
![](http://img.blog.csdn.net/20170804145228358?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

