![](/assets/kubernetes-rbac.png)

在Kubernetes中，授权\(authorization\)是在认证\(authentication\)之后的一个步骤。授权就是决定一个用户\(普通用户或ServiceAccount\)是否有权请求Kubernetes API做某些事情。

之前，Kubernetes中的授权策略主要是ABAC\(Attribute-Based Access Control\)。对于ABAC，Kubernetes在实现上是比较难用的，而且需要Master Node的SSH和根文件系统访问权限，授权策略发生变化后还需要重启API Server。

Kubernetes 1.6中，RBAC\(Role-Based Access Control\)基于角色的访问控制进入Beta阶段。RBAC访问控制策略可以使用kubectl或Kubernetes API进行配置。使用RBAC可以直接授权给用户，让用户拥有授权管理的权限，这样就不再需要直接触碰Master Node。在Kubernetes中RBAC被映射成API资源和操作。

## RBAC API的资源对象 {#ref1}

在Kubernetes 1.6中通过启动参数`--authorization-mode=RBAC.API Overview`为API Server启用RBAC。

使用kubeadm初始化的1.6版本的Kubernetes集群，已经默认为API Server开启了RBAC，可以查看Master Node上API Server的静态Pod定义文件：

```
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep RBAC
    - --authorization-mode=RBAC
```

RBAC API定义了四个资源对象用于描述RBAC中用户和资源之间的连接权限：

* Role
* ClusterRole
* RoleBinding
* ClusterRoleBinding

### Role和ClusterRole {#ref2}

Role是一系列权限的集合。Role是定义在某个Namespace下的资源，在这个具体的Namespace下使用。ClusterRole与Role相似，只是ClusterRole是整个集群范围内使用的。

下面我们使用kubectl打印一下Kubernetes集群中的Role和ClusterRole：

```
kubectl get roles --all-namespaces
NAMESPACE     NAME                                        AGE
kube-public   system:bootstrap-signer-clusterinfo         6d
kube-public   system:controller:bootstrap-signer          6d
kube-system   extension-apiserver-authentication-reader   6d
kube-system   system:controller:bootstrap-signer          6d
kube-system   system:controller:token-cleaner             6d
```

```
kubectl get ClusterRoles
NAME                                           AGE
admin                                          6d
cluster-admin                                  6d
edit                                           6d
flannel                                        5d
system:auth-delegator                          6d
system:basic-user                              6d
system:controller:attachdetach-controller      6d
......
system:kube-aggregator                         6d
system:kube-controller-manager                 6d
system:kube-dns                                6d
system:kube-scheduler                          6d
system:node                                    6d
system:node-bootstrapper                       6d
system:node-problem-detector                   6d
system:node-proxier                            6d
system:persistent-volume-provisioner           6d
view                                           6d
```

可以看到之前创建的这个Kubernetes集群中已经内置或创建很多的Role和ClusterRole。

下面在default命名空间内创建一个名称为pod-reader的Role，role-pord-reader.yaml文件如下：

```

```

```

```

注意RBAC在Kubernetes 1.6还处于Beta阶段，所以API归属在`rbac.authorization.k8s.io`，上面的`apiVersion`为`rbac.authorization.k8s.io/v1beta1`。

下面再给一个ClusterRole的定义文件：

```

```

### RoleBinding和ClusterRoleBinding {#ref3}

RoleBinding把Role绑定到账户主体Subject，让Subject继承Role所在namespace下的权限。ClusterRoleBinding把ClusterRole绑定到Subject，让Subject集成ClusterRole在整个集群中的权限。

账户主体Subject在这里还是叫“用户”吧，包含组group，用户user和ServiceAccount。

```

```

```

```

实际上一个RoleBinding既可以引用相同namespace下的Role；又可以引用一个ClusterRole，RoleBinding引用ClusterRole时用户继承的权限会被限制在RoleBinding所在的namespace下。

```

```

```

```

## Kubernetes中默认的Role和RoleBinding {#ref4}

API Server已经创建一系列ClusterRole和ClusterRoleBinding。这些资源对象中名称以`system:`开头的，表示这个资源对象属于Kubernetes系统基础设施。也就说RBAC默认的集群角色已经完成足够的覆盖，让集群可以完全在 RBAC的管理下运行。修改这些资源对象可能会引起未知的后果，例如对于`system:node`这个ClusterRole定义了kubelet进程的权限，如果这个角色被修改，可能导致kubelet无法工作。

可以使用`kubernetes.io/bootstrapping=rbac-defaults`这个label查看默认的ClusterRole和ClusterRoleBinding：

```

```

```

```

关于这些角色详细的权限信息可以查看[Default Roles and Role Bindings](https://kubernetes.io/docs/admin/authorization/rbac/#default-roles-and-role-bindings)

