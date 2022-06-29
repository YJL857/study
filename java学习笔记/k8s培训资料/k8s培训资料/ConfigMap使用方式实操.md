ConfigMap 功能在 Kubernetes1.2 版本中引入，许多应用程序会从配置文件、命令行参数 或环境变量中读取配 置信息。ConfigMap AP 丨给我们提供了向容器中注入配置信息的机 制，ConfigMap 可以被用来保存单个属性，也 可以用来保存整个配置文件或者 JSON 二进制的对象。

## 1.创建ConfigMap

### 1.1指定配置文件方式创建configMap。

创建配置文件redis.properties文件内容如下：

```
redis.host=192.168.1.2
redis.port=suxianwei
redis.password=123456
```

执行创建configMap命令

```
kubectl create configmap redis-configmap --from-file=redis.properties
```

查看configMap创建结果

```
[root@k8s-master home]# kubectl get configmap
NAME               DATA   AGE
redis-configmap    1      42s
[root@k8s-master home]# kubectl get configmap redis-configmap -o yaml
------
[root@k8s-master home]# kubectl describe cm redis-configmap
```

### 1.2以键值对方式创建configMap

生成yaml文件（configmap-test01.yaml）内容如下：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: suxianwei-configmap-test01
  namespace: default
data:
  special.level: info
  special.type: hello
```

执行创建configMap命令，如下：

```
kubectl apply -f configmap-test001.yaml
```

查看创建configMap结果，如下：

```
[root@k8s-master home]# kubectl get configMap
NAME                         DATA   AGE
suxianwei-configmap-test01   2      13s
[root@k8s-master home]# kubectl get configMap suxianwei-configmap-test01 -o yaml
```

## 2.使用ConfigMap

使用configMap的方式有两种：

​	1.以变量的形式挂载到pod容器中；

​	2.以Volume的形式挂载到pod容器中；

### 2.1以变量的形式挂载到pod容器中

创建pod并挂载ConfigMap，yaml文件pod-configmap-test01.yaml内容如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-test01
spec:
  containers:
  - name: nginx-configmap
    image: nginx:latest
    env:
    - name: LEVEL
      valueFrom:
        configMapKeyRef:
          name: suxianwei-configmap-test01
          key: special.level
    - name: TYPE
      valueFrom:
        configMapKeyRef:
          name: suxianwei-configmap-test01
          key: special.type
  restartPolicy: Never
```

执行pod创建命令，如下：

```
kubectl apply -f pod-configmap-test01.yaml
```

查看pod创建结果，如下：

```
[root@k8s-master home]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
pod-configmap-test01     1/1     Running   0          47s
[root@k8s-master home]# kubectl get pod pod-configmap-test01 -o yaml
```

进入容器查看变量，如下：

```
[root@k8s-master home]# kubectl exec -it pod-configmap-test01 bash
root@pod-configmap-test01:/# echo $LEVEL
info
root@pod-configmap-test01:/# echo $TYPE
hello
```

### 2.2  以Volume的形式挂载到pod容器中   

创建pod并挂载ConfigMap，yaml文件configmap-test02.yaml内容如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-test02
spec:
  containers:
  - name: busybox-configmap-test02
    image: busybox:latest
    command: [ "/bin/sh","-c","cat /var/configmap-test01/redis.properties" ]
    volumeMounts:
    - mountPath: /var/configmap-test01
      name: configmap-test01
  volumes:
  - name: configmap-test01
    configMap:
      name: redis-configmap
  restartPolicy: Never
```

执行创建configMap命令，如下：

```
kubectl apply -f configmap-test02.yaml
```

查看创建执行结果（因为已经是Completed状态，无法进入，查看一下日志）：

```
[root@k8s-master home]# kubectl get pod
NAME                     READY   STATUS      RESTARTS   AGE
pod-configmap-test02     0/1     Completed   0          66s
[root@k8s-master home]# kubectl logs pod-configmap-test02
redis.host=192.168.1.2
redis.port=suxianwei
redis.password=123456
```

将configmap-test01.yaml内容command去掉，Pod在running状态，可以进入容器查看，如下：

```
[root@k8s-master home]# kubectl exec -it pod-configmap-test02 bash
root@pod-configmap-test02:/# cd /var/configmap-test02/
root@pod-configmap-test02:/var/configmap-test02# ls -l
lrwxrwxrwx. 1 root root 23 Apr  8 13:17 redis.properties -> ..data/redis.properties
root@pod-configmap-test02:/var/configmap-test02# cat redis.properties 
redis.host=192.168.1.2
redis.port=suxianwei
redis.password=123456
```

