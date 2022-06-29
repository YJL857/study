Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP和HTTPS。 Ingress 可以提供负载均衡、SSL 和基于名称的虚拟托管。   必须具有 ingress 控制器【例如 ingress-nginx】才能满足 Ingress 的要求。仅创建 Ingress 资源无效。Ingress 公开了从集群外部到集群内 services 的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。Ingress 不会公开任意端口或协议。若将 HTTP 和 HTTPS 以外的服务公开到 Internet 时，通常使用 Service.Type=NodePort 或者 Service.Type=LoadBalancer 类型的服务。

Ingress与Pod的关系：
  1.pod和ingress通过service进行关联的；
  2.根据不同的访问域名跳转到不同的端口服务中；
  2.ingress作为统一入口，由service关联一组pod。

## 1.创建Pod

使用deployment创建pod，命令如下：

```
kubectl create deployment ingress-nginx-test --image=nginx --replicas=2
```

查看创建pod结果如下：

```
[root@k8s-master home]# kubectl get deployment 
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
ingress-nginx-test   0/2     2            0           13s
[root@k8s-master home]# kubectl get pod
NAME                                 READY   STATUS    RESTARTS   AGE
ingress-nginx-test-9bd75f687-jrbbr   1/1     Running   0          22s
ingress-nginx-test-9bd75f687-mj9t9   1/1     Running   0          22s

```

## 2.创建service

创建Service并暴漏deployment服务端口，命令如下：

```
kubectl expose deployment ingress-nginx-test --port=80 --target-port=80 --type=NodePort
```

查看创建service结果如下：

```
service/ingress-nginx-test exposed
[root@k8s-master home]# kubectl get svc
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
ingress-nginx-test        NodePort    10.111.57.42    <none>        80:32754/TCP     9s
```

通过Nodeport方式访问：

http://192.168.0.202:32754/

http://192.168.0.203:32754/

http://192.168.0.204:32754/

## 3.创建Ingress

我们采用Ingress nginx controller，其非K8S内置功能，故需要独立部署。

创建Ingress nginx controller的yaml文件（ingress-controller-test.yaml）内容如下：

```
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      hostNetwork: true
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: nginx-ingress-controller
          image: lizhenliang/nginx-ingress-controller:0.30.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 101
            runAsUser: 101
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown

---

apiVersion: v1
kind: LimitRange
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  limits:
  - min:
      memory: 90Mi
      cpu: 100m
    type: Container
```

执行创建Ingress nginx controller的命令，如下：

```
kubectl apply -f ingress-controller-test.yaml
```

查看创建Ingress nginx controller的结果，如下：

```
[root@k8s-master home]# kubectl get namespace
NAME                   STATUS   AGE
ingress-nginx          Active   2m5s
[root@k8s-master home]# kubectl get pod -n ingress-nginx
NAME                                       READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-5dc64b58f-2x2j6   1/1     Running   0          2m50s
[root@k8s-master home]# kubectl get ing
NAME                CLASS    HOSTS                   ADDRESS   PORTS   AGE
ingress-rule-test   <none>   example.ingredemo.com             80      16m
```

## 4.创建访问规则

创建Ingress访问规则yaml文件（ingress-rule-test.yaml）,内容如下：

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-rule-test
spec:
  rules:
  - host: example.ingredemo.com
    http:
      paths:
      - path: /
        backend:
          serviceName: ingress-nginx-test
          servicePort: 80
```

执行创建Ingress访问规则命令，如下：

```
kubectl apply -f ingress-rule-test.yaml
```

查看Ingress的pod所在节点：

```
[root@k8s-master home]# kubectl get pods -n ingress-nginx -o wide
NAME                                       READY   STATUS    RESTARTS   AGE   IP              NODE        NOMINATED NODE   READINESS GATES
nginx-ingress-controller-5dc64b58f-2x2j6   1/1     Running   0          15m   192.168.0.203   k8s-node1   <none>           <none>
```

在所在节点上查看端口监听情况，如下：

```
[root@k8s-node1 ~]# netstat -antp |grep 80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      72607/nginx: master 
tcp6       0      0 :::80                   :::*                    LISTEN      72607/nginx: master 
[root@k8s-node1 ~]# netstat -antp |grep 443   
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      72607/nginx: master 
tcp6       0      0 :::443                  :::*                    LISTEN      72607/nginx: master 
```

### 5.访问验证

在本地电脑hosts文件中增加节点的域名解析，如下：

```
192.168.0.203	example.ingredemo.com
```

在浏览器中访问域名，如下：

http://example.ingredemo.com

