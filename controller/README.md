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