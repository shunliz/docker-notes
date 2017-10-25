# 弹性伸缩 {#弹性伸缩}

弹性伸缩是指适应负载变化，以弹性可伸缩的方式提供资源。

Pod的弹性伸缩就是修改Replication Controller的Pod副本数。可以通过Kubectl scale命令实现。

## 创建Replication Controller {#创建replication-controller}

test-rc.yaml

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: test-rc
spec:
  replicas: 2
  selector:
    app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: test
        command:
          - sleep
          - "3600"
        image: 192.168.121.143:5000/busybox
        imagePullPolicy: IfNotPresent
      restartPolicy: Always

```

创建RC

```
# kubectl create -f test-rc.yaml 

```

查看创建出的Pod

```
# kubectl get pod --selector app=test -o wide

```

![](http://img.blog.csdn.net/20170804152621943?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

### 扩容Pod的数量到4 {#扩容pod的数量到4}

```
# kubectl scale rc test-rc --replicas=4
# kubectl get pod --selector app=test -o wide

```

![](http://img.blog.csdn.net/20170804152839444?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

### 缩容Pod的数量到1 {#缩容pod的数量到1}

```
# kubectl scale rc test-rc --replicas=1
# kubectl get pod --selector app=test -o wide

```

![](http://img.blog.csdn.net/20170804153138688?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

把Pod的数量设为0，即可删除RC关联的所有Pod。

# 滚动升级 {#滚动升级}

滚动升级采用逐步替换的策略。

通过一个例子进行演示，演示应用从V1版本升到V2版本。

## 创建V1版本的RC。 {#创建v1版本的rc}

my-app-v1-rc.yaml

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-v1
spec:
  replicas: 10
  selector:
    app: myapp
    version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        command:
          - sleep
          - "3600"
        image: 192.168.121.143:5000/busybox:v1
        imagePullPolicy: IfNotPresent
      restartPolicy: Always

```

创建RC，查看RC和Pod。

```
# kubectl create -f my-app-v1-rc.yaml
# kubectl get rc myapp-v1 -o wide
# kubectl get pods --selector app=myapp -o wide

```

![](http://img.blog.csdn.net/20170804160559502?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

## 创建V2版本的RC。 {#创建v2版本的rc}

my-app-v2-rc.yaml

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-v2
spec:
  replicas: 10
  selector:
    app: myapp
    version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        command:
          - sleep
          - "3600"
        image: 192.168.121.143:5000/busybox:v2
        imagePullPolicy: IfNotPresent
      restartPolicy: Always

```

开始滚动升级

```
 kubectl rolling-update myapp-v1 -f my-app-v2-rc.yaml --update-period=10s

```

通过–update-period=10s设置每隔10s逐步增加V2版本的RC的Pod副本数量，逐步减少V1版本的RC的Pod副本数量。  
升级完成后，删除V1版本的RC，保留V2版本的RC。  
![](http://img.blog.csdn.net/20170804161105772?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVxaWdhbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

在升级过程中，可以进行回退。如果升级完成，则不可以使用这条指令进行回退。

```
 kubectl rolling-update myapp-v1 -f my-app-v2-rc.yaml --update-period=10s --rollback
```



