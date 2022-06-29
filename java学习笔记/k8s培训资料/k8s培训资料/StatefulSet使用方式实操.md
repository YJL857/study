

## 1.有状态组件和无状态组件区别

### 1.1无状态组件

​	1.所有POD都是相同的；

​	2.没有顺序要求；

​	3.不用考虑在哪个node运行；

​	4.随意进行伸缩和扩展。

### 1.2有状态组件

​	1.无状态不需要考虑的因素，有状态都需要考虑；

​	2.每个POD都是独立的，保持POD的启动顺序和唯一性。

​	3.唯一的网络标识符，不用IP作为网络标识，采用“域名”的方式进行网络标识；

​	4.持久存储；

​	5.有序，比如mysql主从。

## 2.部署有状态应用-nginx

无头service，即：clusterIP:none

### 2.1创建service和statefulset

创建无头service和有状态组件的yaml文件（stateFulSet.yaml），如下：

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

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
  namespace: default
spec:
  serviceName: nginx
  replicas: 3
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

执行创建命令，如下：

```
kubectl apply -f stateFulSet.yaml
```

查看执行结果（无状组件和无头service）：

```
[root@k8s-master home]# kubectl get pod
NAME                                          READY   STATUS    RESTARTS   AGE
nginx-statefulset-0                           1/1     Running   0          118s
nginx-statefulset-1                           1/1     Running   0          94s
nginx-statefulset-2                           1/1     Running   0          76s
[root@k8s-master home]# kubectl get svc
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
nginx                      ClusterIP   None            <none>        80/TCP         3m17s
```

### 2.2验证statefulset

有状态组件的唯一标识，规则如下：

格式：主机名.service名称.命名空间.svc.cluster.local

例如：nginx-statefulset-0.nginx.default.svc.cluster.local

有顺序创建验证，命令如下：

```
[root@k8s-master home]# kubectl get pod
NAME                  READY   STATUS              RESTARTS   AGE
nginx-statefulset-0   0/1     ContainerCreating   0          9s
[root@k8s-master home]# kubectl get pod
NAME                  READY   STATUS              RESTARTS   AGE
nginx-statefulset-0   1/1     Running             0          33s
nginx-statefulset-1   0/1     ContainerCreating   0          16s
[root@k8s-master home]# kubectl get pod
NAME                  READY   STATUS              RESTARTS   AGE
nginx-statefulset-0   1/1     Running             0          35s
nginx-statefulset-1   1/1     Running             0          18s
nginx-statefulset-2   0/1     ContainerCreating   0          1s
```

唯一标识（域名）验证，命令如下：

```
[root@k8s-master home]# kubectl exec -it nginx-statefulset-0 bash
root@nginx-statefulset-0:/# hostname
nginx-statefulset-0
root@nginx-statefulset-0:/# curl nginx-statefulset-0.nginx.default.svc.cluster.local
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

唯一标识（域名）解析验证，命令如下：

```
[root@k8s-master home]# kubectl run -i --tty --image busybox:1.27 dns-test --restart=Never --rm
If you don't see a command prompt, try pressing enter.
/ # nslookup nginx-statefulset-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-statefulset-0.nginx
Address 1: 10.244.2.60 nginx-statefulset-0.nginx.default.svc.cluster.local

/ # nslookup nginx-statefulset-0.nginx.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-statefulset-0.nginx.default.svc.cluster.local
Address 1: 10.244.2.60 nginx-statefulset-0.nginx.default.svc.cluster.local

/ # nslookup nginx-statefulset-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-statefulset-1.nginx
Address 1: 10.244.2.61 nginx-statefulset-1.nginx.default.svc.cluster.local

/ # nslookup nginx-statefulset-1.nginx.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-statefulset-1.nginx.default.svc.cluster.local
Address 1: 10.244.2.61 nginx-statefulset-1.nginx.default.svc.cluster.local

/ # nslookup nginx-statefulset-2.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-statefulset-2.nginx
Address 1: 10.244.2.62 nginx-statefulset-2.nginx.default.svc.cluster.local

/ # nslookup nginx-statefulset-2.nginx.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-statefulset-2.nginx.default.svc.cluster.local
Address 1: 10.244.2.62 nginx-statefulset-2.nginx.default.svc.cluster.local
```

删除POD后自动重启调起，IP可能会变化，但POD名称和唯一标识（域名）不变，验证如下：

```
[root@k8s-master home]# kubectl get pod -o wide
NAME                  READY   STATUS    RESTARTS   AGE     IP            NODE        NOMINATED NODE   READINESS GATES
nginx-statefulset-0   1/1     Running   0          13m     10.244.2.60   k8s-node1   <none>           <none>
nginx-statefulset-1   1/1     Running   0          13m     10.244.2.61   k8s-node1   <none>           <none>
nginx-statefulset-2   1/1     Running   0          12m     10.244.2.62   k8s-node1   <none>           <none>
[root@k8s-master home]# kubectl delete pod nginx-statefulset-1
pod "nginx-statefulset-1" deleted
[root@k8s-master home]# kubectl get pod -o wide
NAME                  READY   STATUS              RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
nginx-statefulset-0   1/1     Running             0          14m   10.244.2.60   k8s-node1   <none>           <none>
nginx-statefulset-1   0/1     ContainerCreating   0          3s    <none>        k8s-node1   <none>           <none>
nginx-statefulset-2   1/1     Running             0          13m   10.244.2.62   k8s-node1   <none>           <none>
[root@k8s-master home]# kubectl get pod -o wide
NAME                  READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
nginx-statefulset-0   1/1     Running   0          14m   10.244.2.60   k8s-node1   <none>           <none>
nginx-statefulset-1   1/1     Running   0          17s   10.244.2.63   k8s-node1   <none>           <none>
nginx-statefulset-2   1/1     Running   0          13m   10.244.2.62   k8s-node1   <none>           <none>
[root@k8s-master home]# kubectl run -i --tty --image busybox:1.27 dns-test --restart=Never --rm
If you don't see a command prompt, try pressing enter.
/ # nslookup nginx-statefulset-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-statefulset-1.nginx
Address 1: 10.244.2.63 nginx-statefulset-1.nginx.default.svc.cluster.local

/ # nslookup nginx-statefulset-1.nginx.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-statefulset-1.nginx.default.svc.cluster.local
Address 1: 10.244.2.63 nginx-statefulset-1.nginx.default.svc.cluster.local

```

若采用了基于sc创建动态存储，可以做如下验证（参考：https://blog.csdn.net/weixin_44729138/article/details/106054025）：

```
[root@k8s-master home]# kubectl get pod -o wide
NAME                  READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
nginx-statefulset-0   1/1     Running   0          89m   10.244.2.60   k8s-node1   <none>           <none>
nginx-statefulset-1   1/1     Running   0          75m   10.244.2.63   k8s-node1   <none>           <none>
nginx-statefulset-2   1/1     Running   0          88m   10.244.2.62   k8s-node1   <none>           <none>
[root@k8s-master home]# kubectl exec -it nginx-statefulset-0 /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@nginx-statefulset-0:/# echo $(hostname) > /usr/share/nginx/html/index.html
root@nginx-statefulset-0:/# exit
[root@k8s-master home]# kubectl exec -it nginx-statefulset-1  bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@nginx-statefulset-1:/# echo $(hostname) > /usr/share/nginx/html/index.html
root@nginx-statefulset-1:/# exit
exit
[root@k8s-master home]# kubectl exec -it nginx-statefulset-2  bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@nginx-statefulset-2:/# echo $(hostname) > /usr/share/nginx/html/index.html
```

切换到任意一个node上进行，

```
[root@k8s-node1 ~]# curl 10.244.2.60
nginx-statefulset-0
[root@k8s-node1 ~]# curl 10.244.2.63
nginx-statefulset-1
[root@k8s-node1 ~]# curl 10.244.2.62
nginx-statefulset-2
[root@k8s-node1 ~]# 
```

