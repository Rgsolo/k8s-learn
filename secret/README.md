# k8s

###### 生成模版yaml文件

```
kubectl run --image=nginx my-deploy -o yaml --dry-run > my-deploy.yaml 
```

###### 手动编写yaml文件

```
apiVersion: apps/v1
#类型：Deployment，Deployment是pod的控制器
kind: Deployment
#元数据
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
    #管理所有携带“app: nginx”标签的pod，并且确保pod个数为两个
      app: nginx
  # 定义两个pod副本
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

###### 运行pod

```
kubectl create -f nginx-deployment.yaml
```

###### 查看运行状态

```
kubectl get pods -l app=nginx
```

###### DEBUG

```
kubectl describe pod nginx-deployment-5754944d6c-gkg72
```

查看events

###### 升级nginx

```
# 修改nginx-deployment.yaml的内容

kubectl apply -f nginx-deployment.yaml
```

###### 删除

```
kubectl delete -f nginx-deployment.yamld
```

##### secret

###### 创建一个secret

```
cat ./username.txt
admin
cat ./password.txt
c1oudc0w!

kubectl create secret generic user --from-file=./username.txt
kubectl create secret generic pass --from-file=./password.txt
```

###### 使用yaml文件创建

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user0: YWRtaW4=
  pass0: MWYyZDFlMmU2N2Rm
```

用户名和密码都经过base64转码

###### 转码操作

```
echo -n 'admin' | base64
YWRtaW4=
echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
```

###### 查看secret

```
kubectl get secrets
NAME           TYPE                                DATA      AGE
user          Opaque                                1         51s
pass          Opaque                                1         51s
```

###### 使用secret

```
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume 
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
```

###### 创建pod

```
kubectl create -f test-projected-volume.yaml
```

###### 验证

```
kubectl exec -it test-projected-volume -- /bin/sh
ls /projected-volume/
user
pass
cat /projected-volume/user
root
cat /projected-volume/pass
1f2d1e2e67df
```

###### 更新secret

- 重写secret的yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user0: YWRtbWlu
  pass0: YWRtbWlu
```

- 更新

```
kubectl apply -f user0Andpass0.yaml
```

- 验证

```
kubectl exec -it test-projected-volume0 -- /bin/sh

cat projected-volume/pass0
admmin
cat projected-volume/user0
admmin
```

像这样通过挂载方式到容器里的 Secret，一旦其对应的 Etcd 里的数据被更新，这些 Volume 里的文件内容，同样也会被更新。其实，这是 kubelet 组件在定时维护这些 Volume。

##### configmap

它与 Secret 的区别在于，ConfigMap 保存的是不需要加密的、应用所需的配置信息。而 ConfigMap 的用法几乎与 Secret 完全相同：你可以使用 kubectl create configmap 从文件或者目录创建 ConfigMap，也可以直接编写 ConfigMap 对象的 YAML 文件。

比如，一个 Java 应用所需的配置文件（.properties 文件），就可以通过下面这样的方式保存在 ConfigMap 里：

```

# .properties文件的内容
$ cat example/ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice

# 从.properties文件创建ConfigMap
$ kubectl create configmap ui-config --from-file=example/ui.properties

# 查看这个ConfigMap里保存的信息(data)
$ kubectl get configmaps ui-config -o yaml
apiVersion: v1
data:
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  name: ui-config
  ...
```

