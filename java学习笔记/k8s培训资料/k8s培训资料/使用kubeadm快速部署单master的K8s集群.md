
kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```

## 1. 安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 可以访问外网，需要拉取镜像，如果服务器不能上网，需要提前下载镜像并导入节点
- 禁止swap分区

## 2. 准备环境

| 角色   | IP            |
| ------ | ------------- |
| master | 192.168.0.202 |
| node1  | 192.168.0.203 |
| node2  | 192.168.0.204 |

```
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置各主机名
hostnamectl set-hostname k8s-master
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2

# 在master添加hosts
cat >> /etc/hosts << EOF
192.168.0.202 k8s-master
192.168.0.203 k8s-node1
192.168.0.204 k8s-node2
EOF

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# 时间同步
yum install -y chrony
systemctl start chronyd
systemctl enable chronyd
```

## 3. 所有节点安装Docker/kubeadm/kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

### 3.1 安装Docker

docker安装

```
1.更新yum源：
     yum -y update
2.安装依赖环境：
    yum install -y yum-utils device-mapper-persistent-data lvm2
3.yum中添加docker源：
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
4.安装新版containerd.io
	sudo yum install  containerd.io --allowerasing --nobest
5.安装docker社区版
    yum install  docker-ce --allowerasing --nobest
6.开机启动
	systemctl enable docker 
    systemctl start docker
7.查看版本
	docker --version
```

设置镜像加速

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://3o3ptgna.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 3.2 添加阿里云YUM软件源

```
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 3.3 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里可以指定版本号部署：

```
$ yum install -y kubelet kubeadm kubectl
$ systemctl enable kubelet
```

## 4. 部署Kubernetes Master

在192.168.0.202（Master）执行：

```
$ kubeadm init --apiserver-advertise-address=192.168.0.202 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.20.5 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16
```

执行成功会打印一下日志：

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
#在master节点执行
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:
#在node节点执行
kubeadm join 192.168.0.202:6443 --token 4hdyvw.k9g3jjbczcms9i4j \
    --discovery-token-ca-cert-hash sha256:86cadf5eb6199445777f9a275189e8e4a0dc77b3c196e104ee1395a92cfd00dd
```

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

使用kubectl工具：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

## 5. 加入Kubernetes Node

在192.168.0.203/204（Node）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```
kubeadm join 192.168.0.202:6443 --token 4hdyvw.k9g3jjbczcms9i4j \
    --discovery-token-ca-cert-hash sha256:86cadf5eb6199445777f9a275189e8e4a0dc77b3c196e104ee1395a92cfd00dd
```

默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：

```
kubeadm token create --print-join-command
```

## 6. 部署CNI网络插件

默认镜像地址无法访问，sed命令修改为docker hub镜像仓库，如果此国外网址连不上，可参考如下解决方案：

https://blog.csdn.net/weixin_38074756/article/details/109231865

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

部署完成后，执行效果如下：
[root@localhost ~]# kubectl get nodes
NAME         STATUS   ROLES                  AGE   VERSION
k8s-master   Ready    control-plane,master   33m   v1.20.5
k8s-node1    Ready    <none>                 16m   v1.20.5
k8s-node2    Ready    <none>                 16m   v1.20.5
[root@localhost ~]# kubectl get pods -n kube-system
NAME                                 READY   STATUS    RESTARTS   AGE
coredns-7f89b7bc75-6h8qx             1/1     Running   0          35m
coredns-7f89b7bc75-xmrzs             1/1     Running   0          35m
etcd-k8s-master                      1/1     Running   0          35m
kube-apiserver-k8s-master            1/1     Running   0          35m
kube-controller-manager-k8s-master   1/1     Running   0          35m
kube-flannel-ds-ccsjh                1/1     Running   0          6m27s
kube-flannel-ds-cmrtm                1/1     Running   0          6m27s
kube-flannel-ds-tx4k9                1/1     Running   0          6m27s
kube-proxy-5blxv                     1/1     Running   0          19m
kube-proxy-rfst7                     1/1     Running   0          19m
kube-proxy-v8qb6                     1/1     Running   0          35m
kube-scheduler-k8s-master            1/1     Running   0          35m
[root@localhost ~]# 

```

## 7. 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

```
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort
$ kubectl get pod,svc
查看结果如下：
[root@localhost ~]# kubectl get pod,svc
NAME                         READY   STATUS              RESTARTS   AGE
pod/nginx-6799fc88d8-f5gkt   0/1     ContainerCreating   0          23s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        38m
service/nginx        NodePort    10.105.138.19   <none>        80:32300/TCP   9s

```

访问各节点的映射端口（323000）地址：

http://192.168.0.202:32300/

http://192.168.0.203:32300/

http://192.168.0.204:32300/ 

## 8.安装k8s管理web页面

### 8.1安装Kuboard

Kuboard是一款免费的 Kubernetes 图形化管理工具 ，安装命令（参考：https://zhuanlan.zhihu.com/p/97899209）：

```
kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
```

默认安装到kube-system命名空间下，操作结果如下：

```
[root@localhost home]# kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
deployment.apps/kuboard created
service/kuboard created
serviceaccount/kuboard-user unchanged
clusterrolebinding.rbac.authorization.k8s.io/kuboard-user unchanged
serviceaccount/kuboard-viewer unchanged
clusterrolebinding.rbac.authorization.k8s.io/kuboard-viewer unchanged
[root@localhost home]# kubectl get pods -n kube-system
NAME                                 READY   STATUS    RESTARTS   AGE
kuboard-74c645f5df-mppcj             1/1     Running   0          8s
metrics-server-774c4ccbd-fddjs       1/1     Running   0          10m
```

访问地址（默认端口：32567）：

http://192.168.0.203:32567/

获取token命令：

```
echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)
```

### 8.2安装Dashboard

 Dashboard 项目是在 Kubernetes 社区中一个很受欢迎的可视化 Web 界面 （参考：https://blog.csdn.net/networken/article/details/85607593）安装命令如下：

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

执行结果如下：

```
[root@localhost home]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
[root@localhost home]# kubectl -n kubernetes-dashboard get pods
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-7b59f7d4df-5wld4   1/1     Running   0          47s
kubernetes-dashboard-74d688b6bc-p2qd9        1/1     Running   0          47s
[root@localhost home]# kubectl -n kubernetes-dashboard get svc
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.108.107.212   <none>        8000/TCP   2m29s
kubernetes-dashboard        ClusterIP   10.101.117.5     <none>        443/TCP    2m29s
```

暴漏访问命令：

```
kubectl  patch svc kubernetes-dashboard -n kubernetes-dashboard -p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":8443,"nodePort":30443}]}}'
```

执行结果如下：

```
[root@localhost home]# kubectl  patch svc kubernetes-dashboard -n kubernetes-dashboard \
> -p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":8443,"nodePort":30443}]}}'
[root@localhost home]# kubectl -n kubernetes-dashboard get svc
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.108.107.212   <none>        8000/TCP        3m51s
kubernetes-dashboard        NodePort    10.101.117.5     <none>        443:30443/TCP   3m51s

```

创建admin-user的服务账号，并放在kubernetes-dashboard 命名空间下，并将cluster-admin角色绑定到admin-user账户，这样admin-user账户就有了管理员的权限。默认情况下，kubeadm创建集群时已经创建了cluster-admin角色，我们直接绑定即可。

创建dashboard-adminuser.yaml，命令如下：

```
cat > dashboard-adminuser.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard  
EOF
```

创建用户：

```
kubectl apply -f dashboard-adminuser.yaml
```

访问地址（默认端口： 30443 ）：

https://192.168.0.203:30443/#/login

获取token，命令如下：

```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

