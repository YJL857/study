Secret 解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数据暴露 到镜像或者 Pod Spec 中。Secret 可以以 Volume 或者环境变量的方式使用。

## 1.创建Secret

 Secret分为三种类型：

1.Service Account :用来访问 Kubernetes API,由 Kubernetes 自动创建，并且会自动挂 载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中；
2.Opaque : base64 编码格式的 Secret,用来存储密码、密钥等；
3.kubernetes.io/dockerconfigjson ：用来存储私有 docker registry 的认证信息。

### 1.1生成yaml文件

生成yaml文件（secret-test01.yaml）内容如下：

```
apiVersion: v1
kind: Secret
metadata:
  name: suxianwei-secret-test01
  namespace: default
type: Opaque
data:
  username: YWRtaW4=
  password: c3V4aWFud2Vp
```

### 1.2执行创建命令

```
kubectl create -f secret01.yaml
```

### 1.3查看Secret

```
[root@k8s-master home]# kubectl get secret
NAME                      TYPE                                  DATA   AGE
default-token-lfrp5       kubernetes.io/service-account-token   3      3d3h
suxianwei-secret-test01   Opaque                                2      98s
[root@k8s-master home]# kubectl get secret suxianwei-secret-test01 -o yaml
```

## 2.使用Secret

使用secret的方式有两种：

​	1.以变量的形式挂载到pod容器中；

​	2.以Volume的形式挂载到pod容器中；

### 2.1以变量的形式挂载到pod容器中

创建pod并挂载secret，yaml文件secret-test01-pod-1.yaml内容如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-test01
spec:
  containers:
  - name: nginx-seceret-test01
    image: nginx:latest
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: suxianwei-secret-test01
          key: username
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: suxianwei-secret-test01
          key: password
```

执行pod创建命令：

```
kubectl apply -f secret-test01-pod-1.yaml
```

查看pod创建结果：

```
[root@k8s-master home]# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-f5gkt   1/1     Running   0          3d3h
pod-secret-test01        1/1     Running   0          3m42s
[root@k8s-master home]# kubectl get pod pod-secret-test01 -o yaml
```

进入容器查看变量，如下：

```
[root@k8s-master home]# kubectl exec -it pod-secret-test01 bash
root@pod-secret-test01:/# echo $SECRET_USERNAME
admin
root@pod-secret-test01:/# echo $SECRET_PASSWORD
suxianwei
```

### 2.2  以Volume的形式挂载到pod容器中   

创建pod并挂载secret，yaml文件secret-test02-pod-1.yaml内容如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-test02
spec:
  containers:
  - name: nginx-seceret-test02
    image: nginx:latest
    volumeMounts:
    - mountPath: /var/secret-test01
      name: secret-test01
      readOnly: true
  volumes:
  - name: secret-test01
    secret:
      secretName: suxianwei-secret-test01
```

执行pod创建命令：

```
kubectl apply -f secret-test02-pod-1.yaml
```

查看pod创建结果：

```
[root@k8s-master home]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-f5gkt   1/1     Running   0          3d3h
pod-secret-test01        1/1     Running   0          20m
pod-secret-test02        1/1     Running   0          74s
[root@k8s-master home]# kubectl get pod pod-secret-test02 -o yaml
```

进入容器查看挂载信息，如下：

```
[root@k8s-master home]# kubectl exec -it pod-secret-test02 bash        
root@pod-secret-test02:/# cd /var/secret-test01/
root@pod-secret-test02:/var/secret-test01# ls
password  username
root@pod-secret-test02:/var/secret-test01# ls -l
lrwxrwxrwx. 1 root root 15 Apr  7 15:07 password -> ..data/password
lrwxrwxrwx. 1 root root 15 Apr  7 15:07 username -> ..data/username
root@pod-secret-test02:/var/secret-test01# cat password
suxianwei
root@pod-secret-test02:/var/secret-test01# cat username
admin

```

