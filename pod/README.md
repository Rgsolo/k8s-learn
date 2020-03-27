#### pod和container的区别

pod是Kubernetes的最小编排单位，凡是调度，存储，网络，安全相关的属性，基本都是pod相关的。

container是pod属性中的一个普通字段

#### pod中的几个重要字段

 - NodeSelector：是一个供用户将 Pod 与 Node 进行绑定的字段

```
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
```

这样的一个配置，意味着这个 Pod 永远只能运行在携带了“disktype: ssd”标签（Label）的节点上；否则，它将调度失败。

- NodeName：一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。所以，这个字段一般由调度器负责设置，但用户也可以设置它来“骗过”调度器，当然这个做法一般是在测试或者调试的时候才会用到

- HostAliases：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容

```
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

结果：

```
cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```

#### container中的几个重要字段

- **ImagePullPolicy**：定义了镜像拉取的策略。默认值是always，每次创建pod都重新拉取一次镜像。另外，当容器的镜像是类似于 nginx 或者 nginx:latest 这样的名字时，ImagePullPolicy 也会被认为 Always。而如果它的值被定义为 Never 或者 IfNotPresent，则意味着 Pod 永远不会主动拉取这个镜像，或者只在宿主机上不存在这个镜像时才拉取。

- **Lifecycle**：Container Lifecycle Hooks，定义在容器状态发生变化的时候触发的一些动作。

```
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

<u>postStart</u>：在容器启动后，立即执行一些指令。

<u>preStop</u>：容器被杀死之前，执行的一些指令。

#### pod的生命周期

