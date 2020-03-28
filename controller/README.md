## 控制器

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

Deployment的功能是：确保携带了 app=nginx 标签的 Pod 的个数，永远等于 spec.replicas 指定的个数，即 2 个。

kubernetes的控制器都在pkg/controller中：<https://github.com/kubernetes/kubernetes/tree/master/pkg/controller>，Deployment是其中一种。所有的控制器都遵循一个通用编排模式：控制循环。

下面是一段解释这个模式的伪代码：

```
for {
  实际状态 := 获取集群中对象X的实际状态（Actual State）
  期望状态 := 获取集群中对象X的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```

在具体实现中，实际状态来kubernetes集群本身，而期望状态来自于用户提交的yaml



**Deployment 控制器实际操纵的，正是这样的 ReplicaSet 对象，而不是 Pod 对象。**

![img](https://tva1.sinaimg.cn/large/00831rSTgy1gd9qr2sfckj31hc0u0t96.jpg)

ReplicaSet 负责通过“控制器模式”，保证系统中 Pod 的个数永远等于指定的个数

###### 创建控制器

```
kubectl create -f nginx-deployment.yaml --record
```

--record的作用，是记录下你每次操作所执行的命令，以方便后面查看。

###### 检查控制器

```
kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           44s
```

- READY，2/2。已经创建的通过健康检查的副本是2.
- UP-TO-DATE：当前处于最新版本的 Pod 的个数，所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致
- AVAILABLE：当前已经可用的 Pod 的个数，即：既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 Pod 的个数

只有这个 AVAILABLE 字段，描述的才是用户所期望的最终状态

###### 查看控制器状态

```
kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
```

###### 检查ReplicaSet

```
kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-3167673210   3         3         3       20s
```

###### 修改控制器

```
kubectl edit deployment/nginx-deployment
```

修改完之后可以通过describe命令查看events，可以看到deployment修改后，rs是如何一步步改变的

```
kubectl describe deployment nginx-deployment
...
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  40m   deployment-controller  Scaled up replica set nginx-deployment-6f655f5d99 to 2
  Normal  ScalingReplicaSet  17m   deployment-controller  Scaled up replica set nginx-deployment-6f655f5d99 to 3
  Normal  ScalingReplicaSet  12m   deployment-controller  Scaled up replica set nginx-deployment-6f655f5d99 to 4
  Normal  ScalingReplicaSet  82s   deployment-controller  Scaled up replica set nginx-deployment-7448597cd5 to 1
  Normal  ScalingReplicaSet  82s   deployment-controller  Scaled down replica set nginx-deployment-6f655f5d99 to 3
  Normal  ScalingReplicaSet  82s   deployment-controller  Scaled up replica set nginx-deployment-7448597cd5 to 2
  Normal  ScalingReplicaSet  56s   deployment-controller  Scaled down replica set nginx-deployment-6f655f5d99 to 2
  Normal  ScalingReplicaSet  56s   deployment-controller  Scaled up replica set nginx-deployment-7448597cd5 to 3
  Normal  ScalingReplicaSet  55s   deployment-controller  Scaled down replica set nginx-deployment-6f655f5d99 to 1
  Normal  ScalingReplicaSet  55s   deployment-controller  Scaled up replica set nginx-deployment-7448597cd5 to 4
  Normal  ScalingReplicaSet  54s   deployment-controller  Scaled down replica set nginx-deployment-6f655f5d99 to 0
```

###### 再次检查rs

```
kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-6f655f5d99   0         0         0       48m
nginx-deployment-7448597cd5   4         4         4       9m2s
```

deployment滚动更新之后，新生成了一个rs副本，这个副本并不会主动消失，它的作用是什么呢，版本控制。这也是为什么控制器要设计成（deployment-rs-pod）这样三层结构的原因吧我猜测。

### ReplicaSet版本控制

- 修改deployment

```
kubectl edit deployment/nginx-deployment
```
故意写错image的版本
- 查看修改日志

```
kubectl describe deployment/nginx-deployment
···
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  15m    deployment-controller  Scaled up replica set nginx-deployment-6f655f5d99 to 2
  Normal  ScalingReplicaSet  8m41s  deployment-controller  Scaled up replica set nginx-deployment-6f655f5d99 to 3
  Normal  ScalingReplicaSet  8m41s  deployment-controller  Scaled up replica set nginx-deployment-dcff897bd to 1
  Normal  ScalingReplicaSet  8m27s  deployment-controller  Scaled down replica set nginx-deployment-6f655f5d99 to 2
  Normal  ScalingReplicaSet  8m27s  deployment-controller  Scaled up replica set nginx-deployment-dcff897bd to 2
  Normal  ScalingReplicaSet  8m26s  deployment-controller  Scaled down replica set nginx-deployment-6f655f5d99 to 1
  Normal  ScalingReplicaSet  8m26s  deployment-controller  Scaled up replica set nginx-deployment-dcff897bd to 3
  Normal  ScalingReplicaSet  8m25s  deployment-controller  Scaled down replica set nginx-deployment-6f655f5d99 to 0
  Normal  ScalingReplicaSet  4m37s  deployment-controller  Scaled up replica set nginx-deployment-7db786c479 to 1
```

可以看到新创建了一个rs副本，但是创建完之后就停止了

- 查看rs

```
kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-6f655f5d99   0         0         0       16m
nginx-deployment-7db786c479   1         1         0       5m48s
nginx-deployment-dcff897bd    3         3         3       9m52s
```

新创建的这个副本并没有处于ready状态，这个时候我们想回到上个版本

- 回到上个版本

```
kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment rolled back
```

- 回到指定版本

查看所有版本

```
kubectl rollout history deployment/nginx-deployment

deployment.extensions/nginx-deployment
REVISION  CHANGE-CAUSE
1         kubectl create --filename=nginx-deployment.yaml --record=true
3         kubectl create --filename=nginx-deployment.yaml --record=true
4         kubectl create --filename=nginx-deployment.yaml --record=true
```

>老师给的例子里面会出现edit这些动作，但是我的没有，不知道为何。版本2没有是因为被我删除了deployment
>
>> 答案：- - record只针对当前命令有效
>
>```
>$ kubectl rollout history deployment/nginx-deployment
>deployments "nginx-deployment"
>REVISION    CHANGE-CAUSE
>1           kubectl create -f nginx-deployment.yaml --record
>2           kubectl edit deployment/nginx-deployment
>3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91
>```



回到指定版本

```
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

##### 每次更新都会产生一个rs用于回到相关版本，如果不想生成这些rs，该怎么办呢

执行更新	前，先执行暂停deployment的命令

```
$ kubectl rollout pause deployment/nginx-deployment
deployment.extensions/nginx-deployment paused
```

执行更新后，再执行恢复命令

```
$ kubectl rollout resume deploy/nginx-deployment
deployment.extensions/nginx-deployment resumed
```

##### 如果想控制rs的数量，该怎么操作

Deployment 对象有一个字段，叫作 spec.revisionHistoryLimit，就是 Kubernetes 为 Deployment 保留的“历史版本”个数。所以，如果把它设置为 0，你就再也不能做回滚操作了