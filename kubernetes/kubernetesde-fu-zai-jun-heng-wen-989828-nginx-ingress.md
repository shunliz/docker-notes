Kubernetes关于服务的暴露主要是通过NodePort方式，通过绑定minion主机的某个端口,然后进行pod的请求转发和负载均衡，但这种方式下缺陷是

* Service可能有很多个，如果每个都绑定一个node主机端口的话，主机需要开放外围一堆的端口进行服务调用，管理混乱
* 无法应用很多公司要求的防火墙规则

理想的方式是通过一个外部的负载均衡器，绑定固定的端口，比如80,然后根据域名或者服务名向后面的Service ip转发,Nginx很好的解决了这个需求，但问题是如果有新的服务加入，如何去修改Nginx的配置，并且加载这些配置？ Kubernetes给出的方案就是Ingress,Ingress包含了两大主件Ingress Controller和Ingress.

* Ingress解决的是新的服务加入后，域名和服务的对应问题，基本上是一个ingress的对象，通过yaml进行创建和更新进行加载。
* Ingress Controller是将Ingress这种变化生成一段Nginx的配置，然后将这个配置通过Kubernetes API写到Nginx的Pod中，然后reload.

#### 

具体实现如下:

**1.生成一个默认的后端，如果遇到解析不到的URL就转发到默认后端页面  
**

```
[root@k8s-master ingress]# cat default-backend.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    k8s-app: default-http-backend
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        # Any image is permissable as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: gcr.io/google_containers/defaultbackend:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: kube-system
  labels:
    k8s-app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    k8s-app: default-http-backend
```

**2.部署Ingress Controller**

具体文件可以参考官方的

https://github.com/kubernetes/ingress/blob/master/examples/daemonset/nginx/nginx-ingress-daemonset.yaml

这里贴一个我的

```
[root@k8s-master ingress]# cat nginx-ingress-controller.yaml 
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-ingress-lb
  labels:
    name: nginx-ingress-lb
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: nginx-ingress-lb
      annotations:
        prometheus.io/port: '10254'
        prometheus.io/scrape: 'true'
    spec:
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      containers:
      - image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.7
        name: nginx-ingress-lb
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          timeoutSeconds: 1
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: KUBERNETES_MASTER
            value: http://192.168.0.105:8080
        args:
        - /nginx-ingress-controller
        - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
        - --apiserver-host=http://192.168.0.105:8080
```

曾经出现的问题是，启动后pod总是在CrashLoopBack的状态，通过logs一看发现nginx-ingress-controller的启动总是去连接apiserver内部集群ip的443端口，导致因为安全问题不让启动，后来在args里面加入

```
- --apiserver-host=http://192.168.0.105:8080

```

后成功启动.

![](/assets/kube-ingress.png)



