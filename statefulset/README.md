## statefulset

statefulset的设计分为两种：拓扑状态和存储状态

拓扑状态：

> 这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个 Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的 Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。

存储状态

> 这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。

### statefulset—拓扑状态

- headless service

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

这是一个标准的headless service文件，也就是目录中的svc.yaml

它所代理的pod的ip地址都会以下面的DNS记录

```
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

- statefulset

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```

这个文件中指定了serviceName=nginx

创建完上面的service和statefulset之后

```
kubectl get service nginx
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   None         <none>        80/TCP    40h
```

```
kubectl get statefulset web
NAME   READY   AGE
web    2/2     40h
```

```
kubectl get pod -l app=nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7448597cd5-52jvp   1/1     Running   0          52m
nginx-deployment-7448597cd5-6stkq   1/1     Running   0          52m
nginx-deployment-7448597cd5-n7mnw   1/1     Running   0          52m
web-0                               1/1     Running   0          52m
web-1                               1/1     Running   0          52m
```

进入到pod中，查看hostname

```
kubectl exec web-0 -- sh -c "hostname"
web-0
```

```
kubectl exec web-1 -- sh -c "hostname"
web-1
```

以dns的方式访问headless service

```

$ kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.7

$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.2.7
```

### statefulset—拓扑状态

创建pv

```

```

