PersistentVolume（PV）是集群中由管理员配置的一段网络存储。 它是集群中的资源，就像节点是集群资源一样。 PV是容量插件，如Volumes，但其生命周期独立于使用PV的任何单个pod。
PersistentVolumeClaim（PVC）是由用户进行存储的请求。它类似于pod。Pod消耗节点资源，PVC消耗PV资源。

PV：持久化存储，对存储资源进行抽象，对外提供可以调用的地方（生产者）。

PVC：用于调用，不需要关心内部实现细节（消费者）。

注：PV、PVC和NFS没有本质的区别，只不过是PV的是由专业的管理员进行管理，从而增强了安全，使用人员只需要使用即可。

## 1.创建无状态组件和PVC

生成创建无状态组件和PVC的yaml文件deployment-pvc.yaml，内容如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dep1
spec:
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
        image: nginx
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
      volumes:
      - name: wwwroot
        persistentVolumeClaim:
          claimName: my-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

执行创建命令并查看，如下：

```
[root@k8s-master pv]# kubectl apply -f deployment-pvc.yaml 
deployment.apps/nginx-dep1 created
persistentvolumeclaim/my-pvc created
[root@k8s-master pv]# kubectl get deployment
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
nginx-dep1   0/3     3            0           7s
[root@k8s-master pv]# kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
nginx-dep1-69f5bb95b-mngcb   0/1     Pending   0          14s
nginx-dep1-69f5bb95b-s7dw4   0/1     Pending   0          14s
nginx-dep1-69f5bb95b-tfqkh   0/1     Pending   0          14s
[root@k8s-master pv]# kubectl get pvc
NAME                STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc              Bound     my-pv    2Gi        RWX                           4m50s
mysql-pvc-mysql-0   Pending                                                     9h
```

## 2.创建PV

生成创建PV的yaml文件my-pv.yaml，内容如下：

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /data/nfs
    server: 192.168.0.201
```

执行创建命令并查看，如下：

```
[root@k8s-master pv]# kubectl apply -f my-pv.yaml 
persistentvolume/my-pv created
[root@k8s-master pv]# kubectl get pv,pvc
NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
persistentvolume/my-pv   2Gi        RWX            Retain           Bound    default/my-pvc                           55s

NAME                                      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/my-pvc              Bound     my-pv    2Gi        RWX                           3m58s
persistentvolumeclaim/mysql-pvc-mysql-0   Pending   
```

### 3.验证

进入pod，查看挂载路径，仍可以看到之前NFS操作遗留的index.html文件，如下：

```
[root@k8s-master pv]# kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
nginx-dep1-69f5bb95b-mngcb   1/1     Running   0          11m
nginx-dep1-69f5bb95b-s7dw4   1/1     Running   0          11m
nginx-dep1-69f5bb95b-tfqkh   1/1     Running   0          11m
[root@k8s-master pv]# kubectl exec -it nginx-dep1-69f5bb95b-tfqkh bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@nginx-dep1-69f5bb95b-tfqkh:/# cd /usr/share/nginx/html
root@nginx-dep1-69f5bb95b-tfqkh:/usr/share/nginx/html# ls
index.html
```