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

[https://github.com/kubernetes/ingress/blob/master/examples/daemonset/nginx/nginx-ingress-daemonset.yaml](https://github.com/kubernetes/ingress/blob/master/examples/daemonset/nginx/nginx-ingress-daemonset.yaml)

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

**3.配置ingress**

```
[root@k8s-master ingress]# cat dashboard-weblogic.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-weblogic-ingress
  namespace: kube-system
spec:
  rules:
  - host: helloworld.eric
    http:
      paths:
      - path: /console
        backend:
          serviceName: helloworldsvc 
          servicePort: 7001
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 80
```

理解如下:

* host指虚拟出来的域名，具体地址\(我理解应该是Ingress-controller那台Pod所在的主机的地址\)应该加入/etc/hosts中,这样所有去helloworld.eric的请求都会发到nginx
* path:/console匹配后面的应用路径
* servicePort主要是定义服务的时候的端口，不是NodePort.
* path:/ 匹配后面dashboard应用的路径,以前通过访问master节点8080/ui进入dashboard的，但dashboard其实是部署在minion节点中，实际是通过某个路由语句转发过去而已，dashboard真实路径如下:

![](/assets/kube-ingress2.png)

而yaml文件是

```
[root@k8s-master ~]# cat kubernetes-dashboard.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
# Keep the name in sync with image version and
# gce/coreos/kube-manifests/addons/dashboard counterparts
  name: kubernetes-dashboard-latest
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
        version: latest
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: kubernetes-dashboard
        image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.5.1
        resources:
          # keep request = limit to keep this container in guaranteed class
          limits:
            cpu: 100m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 50Mi
        ports:
        - containerPort: 9090
        args:
         -  --apiserver-host=http://192.168.0.105:8080
        livenessProbe:
          httpGet:
            path: /
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
---
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    k8s-app: kubernetes-dashboard
  ports:
  - port: 80
    targetPort: 9090
```

所以访问192.168.51.5:9090端口就会出现dashboard

![](/assets/kube-ingress3.png)

**4.测试 **

Ok,一切就绪，装逼开始

访问[http://helloworld.eric/console](http://helloworld.eric/console)

访问[http://helloword.eric/](http://helloword.eric/)    出现dashboard



**5.配置TLS SSL访问**

TLS的配置相当于WebLogic中证书的配置，配置过程如下

* 证书生成

```
# 生成 CA 自签证书
mkdir cert && cd cert
openssl genrsa -out ca-key.pem 2048
openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca"

# 编辑 openssl 配置
cp /etc/pki/tls/openssl.cnf .
vim openssl.cnf

# 主要修改如下
[req]
req_extensions = v3_req # 这行默认注释关着的 把注释删掉
# 下面配置是新增的
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = helloworld.eric
#DNS.2 = kibana.mritd.me

# 生成证书
openssl genrsa -out ingress-key.pem 2048
openssl req -new -key ingress-key.pem -out ingress.csr -subj "/CN=helloworld.eric" -config openssl.cnf
openssl x509 -req -in ingress.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out ingress.pem -days 365 -extensions v3_req -extfile openssl.cnf
```

需要注意的是DNS需要修改成自己的host名,然后在配置csr证书请求的时候需要将域名或者访问名带入subj,比如

```
-subj "/CN=helloworld.eric" 
```

* 创建secret

```
kubectl create secret tls ingress-secret --namespace=kube-system --key cert/ingress-key.pem --cert cert/ingress.pem 
```

修改Ingress文件启用证书

```
[root@k8s-master ingress]# cat tls-weblogic.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-weblogic-ingress
  namespace: kube-system
spec:
  tls:
  - hosts:
    - helloworld.eric
    secretName: ingress-secret
  rules:
  - host: helloworld.eric
    http:
      paths:
      - path: /console
        backend:
          serviceName: helloworldsvc 
          servicePort: 7001
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 80
```

 测试

 然后访问helloworld.eric/console,会自动转到https页面，同时查看证书并加入授信列表，可见portal



