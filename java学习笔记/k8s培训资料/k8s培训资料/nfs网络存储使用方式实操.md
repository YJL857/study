 NFS(network file system)网络文件系统，类似Windows中的文件夹共享 。

## 1.部署NFS

nfs分为服务端和客户端，在nfs服务器上安装服务端，在k8s的各node上安装nfs客户端。

### 1.1nfs服务器安装nfs

在NFS服务器上（IP:192.168.0.201）使用yum安装nfs，命令如下：

```
yum install -y nfs-utils
```

### 1.2nfs服务器设置挂载路径

创建挂载路径并编辑nfs配置，如下：

```
[root@localhost ~]# mkdir /data/nfs
[root@localhost ~]# vi /etc/exports
```

设置挂载路径到/data/nfs并设置权限：

```
/data/nfs *(rw,no_root_squash)
```

/data/nfs 为共享目录

*可以为一个网段，一个IP，也可以是域名，域名支持通配符 如: *.qq.com
rw：read-write，可读写；
ro：read-only，只读；
sync：文件同时写入硬盘和内存；
async：文件暂存于内存，而不是直接写入内存；
no_root_squash：NFS客户端连接服务端时如果使用的是root的话，那么对服务端分享的目录来说，也拥有root权限。显然开启这项是不安全的。
root_squash：NFS客户端连接服务端时如果使用的是root的话，那么对服务端分享的目录来说，拥有匿名用户权限，通常他将使用nobody或nfsnobody身份；
all_squash：不论NFS客户端连接服务端时使用什么用户，对服务端分享的目录来说都是拥有匿名用户权限；
anonuid：匿名用户的UID值，可以在此处自行设定。
anongid：匿名用户的GID值

另外，如果访问不到，还需要关闭防火墙，命令如下：

```
systemctl stop firewalld.service
systemctl disable firewalld.service
firewall-cmd --state
```

启动服务，命令如下：

```
先为rpcbind和nfs做开机启动：    
systemctl enable rpcbind.service    
systemctl enable nfs-server.service    
然后分别启动rpcbind和nfs服务：   
systemctl start rpcbind.service    
systemctl start nfs-server.service
```

验证挂载路径，命令如下：

```
[root@localhost init.d]# showmount -e 192.168.0.201
Export list for 192.168.0.201:
/data/nfs *
```

### 1.3在各node节点上安装nfs客户端

在node1（IP:192.168.0.203）和node2（IP:192.168.0.204）上均安装nfs，命令如下：

```
yum install -y nfs-utils
```

启动nfs客户端并开启启动，如下：

```
systemctl enable rpcbind.service
systemctl start rpcbind.service
```

检查nfs服务器端是否有目录成功共享，命令如下：

```
[root@localhost init.d]# showmount -e 192.168.0.201
Export list for 192.168.0.201:
/data/nfs *
```

## 2.部署应用并使用NFS

创建生成deployment的yaml文件nfs-nginx.yaml，内容如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dep-nfs
spec:
  replicas: 1
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
          nfs:
            server: 192.168.0.201
            path: /data/nfs
```

执行创建命令，并查看结果：

```
[root@k8s-master pv]# kubectl apply -f nfs-nginx.yaml 
deployment.apps/nginx-dep-nfs created
[root@k8s-master pv]# kubectl get deployment
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
nginx-dep-nfs   1/1     1            0           18s
[root@k8s-master pv]# kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
nginx-dep-nfs-68c5b55576-bppz4   1/1     Running   0          25m
```

若创建后pod一直运行不起来，可以查看启动过程，命令如下：

```
[root@k8s-master pv]# kubectl describe pod nginx-dep-nfs-68c5b55576-bppz4
```

## 3.验证NFS

### 3.1查看nfs共享目录

首先，进入pod查看挂载目录，无任何文件，如下：

```
[root@k8s-master pv]# kubectl exec -it nginx-dep-nfs-68c5b55576-bppz4 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@nginx-dep-nfs-68c5b55576-bppz4:/# cd /usr/share/nginx/html
root@nginx-dep-nfs-68c5b55576-bppz4:/usr/share/nginx/html# ls
```

其次，在nfs服务共享目录（/data/nfs）下创建index.html，然后在查看pod挂载目录，如下：

```
root@nginx-dep-nfs-68c5b55576-bppz4:/usr/share/nginx/html# ls
index.html
```

### 3.2暴露服务并访问

将deployment中pod的80端口暴露出去，如下：

```
[root@k8s-master pv]# kubectl expose deployment nginx-dep-nfs --port=80 --target-port=80 --type=NodePort
service/nginx-dep-nfs exposed
[root@k8s-master pv]# kubectl get svc
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
nginx-dep-nfs   NodePort    10.97.15.24   <none>        80:32583/TCP   33s
```

在浏览器中访问，地址如下：

http://192.168.0.202:32583/

http://192.168.0.203:32583/

http://192.168.0.204:32583/