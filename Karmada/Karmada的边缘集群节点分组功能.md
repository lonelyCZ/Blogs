# Karmada的边缘集群节点分组功能
## 1. Karmada的思想在边缘分组中的应用
在边缘计算的场景下，一个云端APIServer可以支持的边缘节点数量大幅度增加，用户希望通过一个APIServer就能管理众多的边缘节点。例如，一家公司有众多的子公司，客户不想在每个子公司都安装一个APIServer，而是只向一个子公司分配数个边缘节点，就可以完成其业务需求。所以实现边缘节点分组管理机制是有必要的。  

Karmada是一个多云多集群的管理系统，管理的每一个子集群都有一个APIServer，但是其也可以扩展为边缘集群的分组管理，如在Cluster中添加有关分组的信息，使一个边缘分组成为Virtual Cluster，这样可以利用Karmada的编排调度能力来实现边缘结点的分组管理。

### 1.1. Virtual Cluster在Karmada中的架构图
![image](https://i.bmp.ovh/imgs/2021/09/40cfa095ffc0343a.png)

### 1.2.Karmada中支持的操作
1. 可以将一个APIServer映射称为多个Member Cluster，只需要Member Cluster的name不同即可。(Push模式)

### 1.3.资源转换流程
![image](https://i.bmp.ovh/imgs/2021/09/79ac4b279da1577a.png)

## 2. Karmada现有机制实现节点分组功能
### 2.1. 对同一个apiserver创建两个Name不同的Context
在/root/.kube/karmada.config中添加一个name为member2的context，但member2与member1指向同一个member cluster的APIServer。
```yaml
- context:
    cluster: kind-member1
    user: kind-member1
  name: member1
- context:
    cluster: kind-member1
    user: kind-member1
  name: member2
```

### 2.2. 使用karmadactl join添加member2集群
执行karmadactl join命令
```bash
$ karmadactl join member2 --cluster-kubeconfig=$HOME/.kube/karmada.config
```
member2成为karmada管理的member cluster。但member1和member2其实是指向同一个集群。
```bash
[root@localhost karmadactl]# kubectl get cluster
NAME      VERSION   MODE   READY   AGE
member1   v1.21.1   Push   True    178m
member2   v1.21.1   Push   True    112s
```

### 2.3. 利用PropagationPolicy和OverridePolicy实现边缘集群分组
member1和member2实际上是同一个集群
```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames:
        - member1
        - member2

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

如果按以上yaml直接部署，虽然karmada层面看到的是在两个子集群中创建了deployment资源，但是实际上子集群中只有一个deployment资源，因为deployment的名称都相同。

**创建OverridePolicy实现分组管理**

- 分别为member1和member2创建对应的OverridePolicy
- 分别修改两个OverridePolicy中Deployment资源的`namespace`，避免其重名
- 分别为两个OverridePolidy中Deployment资源添加`nodeSelector`标签，以实现将pod部署到两个不同的分组之中

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: member1-override
  namespace: default
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
  targetCluster:
    clusterNames:
      - member1
  overriders:
    plaintext:
      - path: "/metadata/namespace"
        operator: replace
        value: "member1"
      - path: "/spec/template/spec/nodeSelector"
        operator: add
        value:
          location: beijing

---
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: member2-override
  namespace: default
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
  targetCluster:
    clusterNames:
      - member2
  overriders:
    plaintext:
      - path: "/metadata/namespace"
        operator: replace
        value: "member2"
      - path: "/spec/template/spec/nodeSelector"
        operator: add
        value:
          location: hangzhou
```

### 2.4. 集群中的资源转换情况
**Karmada中的资源**
- work：分别为member1和member2创建了work，但是work名并不相同
```bash
[root@localhost yaml]# kubectl get work -A
NAMESPACE            NAME               AGE
karmada-es-member1   nginx-86bcb79f89   4h2m
karmada-es-member2   nginx-7dbb9f898b   4h2m
```

**子集群中的资源**
- deployment：分别在不同的namespace中创建了nginx，并且在template中加上了`nodeSelector`字段，分别添加了对应的标签`location: beijing`和`location: hangzhou`
```bash
[root@localhost yaml]# kubectl get deploy -A
NAMESPACE            NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
member1              nginx                    0/1     1            0           4h8m
member2              nginx                    0/1     1            0           4h8m
```
```bash
[root@localhost yaml]# kubectl get deploy nginx -n member1 -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
spec:
  ...
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
      nodeSelector:
        location: beijing
      ...

[root@localhost yaml]# kubectl get deploy nginx -n member2 -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
spec:
  ...
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
      nodeSelector:
        location: hangzhou
      ...
```

至此，只需要给子集群中的Node分别打赏对应的`location`标签，即可实现节点的分组管理


## 3. 简化Karmada的节点分组部署分组步骤
1. 在每个Cluster对象中添加labels，用于表示该分组匹配哪些节点
2. 将ResourceBinding转换为Work时，自动将对应Cluster中的labels注入到资源的**NodeSelector**字段中。
3. **因为边缘集群只有一个API-Server，但需要接收同一Resource转换的多个Work，所以需要对Resource在边缘集群中，需要对Resource资源进行隔离。(Resource name不重复或者namespace不相同)**