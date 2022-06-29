Service 是 Kubernetes 最核心概念，通过创建 Service,可以为一组具有相同功能的容器应 用提供一个统一的入口地 址，并且将请求负载分发到后端的各个容器应用上。  

## 1.Service作用

1.防止POD失联（服务发现）； 

2.定义一组Pod访问策略（负载均衡）；

pod与service是根据pod的labels.app与service的selector.app建立关联的。

## 2.Service类型

1.ClusterIP：集群内部访问使用；  

2.NodePort：对外访问应用使用，  每个node都开启了此端口并可以访问 ；  

3.LoadBalance：对外访问应用使用，可以连接公有云，实现负载均衡。

## 3.创建Service

### 3.1创建有labels的pod

创建pod的yaml文件（pod-service-test01.yaml）内容如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-service-test02
  labels: 
    app: pod-labels-test
spec:
  containers:
  - name: tomcat-service
    image: tomcat:latest
  restartPolicy: Never
```

执行创建pod命令，如下：

```
kubectl apply -f pod-service-test01.yaml
```

查看创建pod结果，如下：

```
[root@k8s-master home]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
pod-service-test02       1/1     Running   0          2m21s
[root@k8s-master home]# kubectl get pod pod-service-test02 -o yaml
```

### 3.2创建有selector的service

创建service的yaml文件（service-test01.yaml）内容如下：

```
apiVersion: v1
kind: Service
metadata:
  name: service-nodeport-test01
spec:
  ports:
  - nodePort: 30001
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: pod-labels-test
  type: NodePort
status:
  loadBalancer: {}
```

执行创建service命令，如下：

```
kubectl apply -f service-test01.yaml
```

查看创建service结果，如下：

```
[root@k8s-master home]# kubectl get svc
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service-nodeport-test01   NodePort    10.100.108.51   <none>        8080:30001/TCP   2m47s
```

在浏览器中访问tomcat，各节点访问地址如下：

http://192.168.0.202:30001/

http://192.168.0.203:30001/

http://192.168.0.204:30001/

