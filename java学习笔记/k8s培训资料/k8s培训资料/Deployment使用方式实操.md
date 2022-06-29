Deployment 是 Kubenetes v1.2 引入的新概念，引入的目的是为了更好的解决 Pod 的编排 问题，Deployment 内部使用了 Replica Set 来实现，用途如下：   

1.部署无状态应用；  

2.管理Pod和ReplicaSet；  

3.部署、滚动升级等功能。

## 1.创建deployment并部署应用

生成yaml文件suxianwei-deployment-test01.yaml，命令如下：

```
kubectl create deployment suxianwei-deployment-test01 --image=nginx --dry-run -o yaml > suxianwei-deployment-test01.yaml
```

按实际需求修改yaml文件suxianwei-deployment-test01.yaml，内容如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: suxianwei-deployment-test01
  name: suxianwei-deployment-test01
spec:
  replicas: 2
  selector:
    matchLabels:
      app: suxianwei-deployment-test01
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: suxianwei-deployment-test01
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

执行创建deployment命令，如下：

```
kubectl apply -f suxianwei-deployment-test01.yaml
```

查看创建deployment结果，如下：

```
[root@k8s-master home]# kubectl get deployment
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
suxianwei-deployment-test01   2/2     2            2           77s
[root@k8s-master home]# kubectl get pod
NAME                                           READY   STATUS    RESTARTS   AGE
suxianwei-deployment-test01-646c96d4b9-6r4lw   1/1     Running   0          81s
suxianwei-deployment-test01-646c96d4b9-ph74j   1/1     Running   0          81s
```

## 2.创建Service暴露应用端口

生成yaml文件suxianwei-service-test01.yaml，命令如下：

```
kubectl expose deployment suxianwei-deployment-test01 --port=80 --type=NodePort --target-port=80 --name=suxianwei-service-test01 -o yaml >suxianwei-service-test01.yaml
```

按实际需求修改yaml文件suxianwei-service-test01.yaml，内容如下：

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2021-04-12T14:15:48Z"
  labels:
    app: suxianwei-deployment-test01
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .: {}
          f:app: {}
      f:spec:
        f:externalTrafficPolicy: {}
        f:ports:
          .: {}
          k:{"port":80,"protocol":"TCP"}:
            .: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
        f:selector:
          .: {}
          f:app: {}
        f:sessionAffinity: {}
        f:type: {}
    manager: kubectl-expose
    operation: Update
    time: "2021-04-12T14:15:48Z"
  name: suxianwei-service-test01
  namespace: default
  resourceVersion: "288222"
  uid: ab182273-b8ff-496b-b1f4-00040106b014
spec:
  clusterIP: 10.105.122.48
  clusterIPs:
  - 10.105.122.48
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30610
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: suxianwei-deployment-test01
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

执行创建service命令，如下：

```
kubectl apply -f suxianwei-service-test01.yaml
```

查看创建deployment结果，如下：

```
[root@k8s-master home]# kubectl get svc
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
suxianwei-service-test01   NodePort    10.105.122.48   <none>        80:30610/TCP     3s
```

## 3.访问应用验证

在浏览器中访问任意一个node的映射端口，如下：

http://192.168.0.202:30610/

http://192.168.0.203:30610/

http://192.168.0.204:30610/

## 4.升级回滚

可以使用命令方式和修改yaml文件的方式进行升级。

### 4.1升级应用

1.命令方式：将deployment升级指定版本的镜像，命令如下：

```
kubectl set image deployment suxianwei-deployment-test01 nginx=nginx:1.15
```

2.yaml文件方式：将suxianwei-service-test01.yaml文件中镜像改为“nginx:1.15”，如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: suxianwei-deployment-test01
  name: suxianwei-deployment-test01
spec:
  replicas: 2
  selector:
    matchLabels:
      app: suxianwei-deployment-test01
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: suxianwei-deployment-test01
    spec:
      containers:
      - image: nginx:1.15
        name: nginx
        resources: {}
status: {}
```

然后执行如下命令，即可：

```
kubectl apply -f suxianwei-service-test01.yaml
```

查看升级状态，命令如下：

```
[root@k8s-master home]# kubectl rollout status deployment suxianwei-deployment-test01
deployment "suxianwei-deployment-test01" successfully rolled out
```

查看升级后deployment信息，如下：

```
[root@k8s-master home]# kubectl get deployment suxianwei-deployment-test01 -o yaml
```

### 4.2回滚应用

查看历史版本，命令如下：

```
[root@k8s-master home]# kubectl rollout history deployment suxianwei-deployment-test01
deployment.apps/suxianwei-deployment-test01 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

回滚到上一个版本，命令如下：

```
[root@k8s-master home]# kubectl rollout undo deployment suxianwei-deployment-test01
deployment.apps/suxianwei-deployment-test01 rolled back
```

回滚到指定版本，命令如下：

```
[root@k8s-master home]# kubectl rollout undo deployment suxianwei-deployment-test01 --to-revision=2
deployment.apps/suxianwei-deployment-test01 rolled back
```

查看回归状态，命令如下：

```
[root@k8s-master home]# kubectl rollout status deployment suxianwei-deployment-test01
deployment "suxianwei-deployment-test01" successfully rolled out
```

## 5.弹性伸缩

将deployment的副本数设置为指定数，命令如下：

```
[root@k8s-master home]# kubectl scale deployment suxianwei-deployment-test01 --replicas=10
deployment.apps/suxianwei-deployment-test01 scaled
```

查看伸缩后的deployment信息，如下：

```
[root@k8s-master home]# kubectl get deployment suxianwei-deployment-test01 -o yaml
[root@k8s-master home]# kubectl get pod
NAME                                         READY   STATUS              RESTARTS   AGE
suxianwei-deployment-test01-c554879d-5gvc6   0/1     ContainerCreating   0          78s
suxianwei-deployment-test01-c554879d-6hgcn   1/1     Running             0          6m39s
suxianwei-deployment-test01-c554879d-dx6z2   1/1     Running             0          78s
suxianwei-deployment-test01-c554879d-g5xgq   1/1     Running             0          78s
suxianwei-deployment-test01-c554879d-jb565   1/1     Running             0          78s
suxianwei-deployment-test01-c554879d-k5sjh   1/1     Running             0          78s
suxianwei-deployment-test01-c554879d-lf5lb   0/1     ContainerCreating   0          78s
suxianwei-deployment-test01-c554879d-tjfn4   1/1     Running             0          6m22s
suxianwei-deployment-test01-c554879d-tzb4r   0/1     ContainerCreating   0          78s
suxianwei-deployment-test01-c554879d-vzgjk   1/1     Running             0          78s
```

