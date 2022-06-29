# K8s资源编排

**如何快速编写yaml文件**

1. **使用kubectl create命令生成yaml文件**

   kubectl create deployment web --image=nginx -o yaml --dry-run >nginx.yaml

2. **使用kubectl get 命令导出yaml文件**

   kubectl get deploy nginx -o=yaml --export > nginx2.yaml

