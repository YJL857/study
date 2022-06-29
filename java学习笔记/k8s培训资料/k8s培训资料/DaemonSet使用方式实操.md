服务守护进程，它的主要作用是在Kubernetes集群的所有节点中运行我们部署的守护进程，相当于在集群节点上分别部署Pod副本，如果有新节点加入集群，Daemonset会自动的在该节点上运行我们需要部署的Pod副本，相反如果有节点退出集群，Daemonset也会移除掉部署在旧节点的Pod副本。

DaemonSet的Pod有三个主要特征：

1. 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；
2. 每个节点上只有一个这样的 Pod 实例；
3. 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。

## 1.创建守护进程采集node主机日志

### 1.1创建守护进程

生成创建守护进程的yaml文件deamonSet-test.yaml，内容如下（主机目录）：

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-test 
  labels:
    app: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: logs
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: varlog
          mountPath: /tmp/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

执行创建命令并查看结果：

```
[root@k8s-master home]# kubectl apply -f deamonSet-test.yaml 
daemonset.apps/ds-test created
[root@k8s-master home]# kubectl get pod
NAME            READY   STATUS              RESTARTS   AGE
ds-test-2h6sx   0/1     ContainerCreating   0          6s
ds-test-nmgqs   0/1     ContainerCreating   0          6s
[root@k8s-master home]# kubectl get pod -o wide
NAME            READY   STATUS              RESTARTS   AGE   IP       NODE        NOMINATED NODE   READINESS GATES
ds-test-2h6sx   0/1     ContainerCreating   0          16s   <none>   k8s-node1   <none>           <none>
ds-test-nmgqs   0/1     ContainerCreating   0          16s   <none>   k8s-node2   <none>           <none>
```

### 1.2验证结果

在各node中创建文件，并查看：

```
[root@k8s-node1 ~]# cd /var/log/
[root@k8s-node1 log]# touch node1.txt
[root@k8s-node1 log]# ll
-rw-r--r--  1 root   root         0 4月  18 03:53 node1.txt

[root@k8s-node2 ~]# cd /var/log/
[root@k8s-node2 log]# touch node2.txt
[root@k8s-node2 log]# ll
-rw-r--r--  1 root   root         0 4月  18 03:57 node2.txt
```

进入部署在node上的pod，并查看日志，如下：

```
[root@k8s-master home]# kubectl get pod -o wide
NAME            READY   STATUS              RESTARTS   AGE   IP       NODE        NOMINATED NODE   READINESS GATES
ds-test-2h6sx   0/1     ContainerCreating   0          16s   <none>   k8s-node1   <none>           <none>
ds-test-nmgqs   0/1     ContainerCreating   0          16s   <none>   k8s-node2   <none>           <none>

[root@k8s-master home]# kubectl exec -it ds-test-2h6sx bash
root@ds-test-2h6sx:/# cd /tmp/log
root@ds-test-2h6sx:/tmp/log# ls -l
-rw-r--r--  1 root root        0 Apr 18 07:53 node1.txt

[root@k8s-master home]# kubectl exec -it ds-test-nmgqs bash
root@ds-test-nmgqs:/# cd /tmp/log
root@ds-test-nmgqs:/tmp/log# ls -l
-rw-r--r--  1 root root        0 Apr 18 07:57 node2.txt

```

