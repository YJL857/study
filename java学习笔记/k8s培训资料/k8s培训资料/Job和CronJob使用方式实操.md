Job负责处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个Pod成功结束。而CronJob则就是在Job上加上了时间调度。 其实job和cronjob是一样的功能，只不过cronjob添加了定时任务功能。 

## 1.创建一次性任务

生产创建JOB的yaml文件job-test.yaml，内容如下：

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

执行创建命令，并查看：

```
[root@k8s-master home]# kubectl apply -f job-test.yaml 
job.batch/pi created
[root@k8s-master home]# kubectl get pod
NAME            READY   STATUS              RESTARTS   AGE
ds-test-2h6sx   1/1     Running             0          39m
ds-test-nmgqs   1/1     Running             0          39m
pi-lgwjj        0/1     ContainerCreating   0          6s
[root@k8s-master home]# kubectl get pod
NAME            READY   STATUS      RESTARTS   AGE
ds-test-2h6sx   1/1     Running     0          65m
ds-test-nmgqs   1/1     Running     0          65m
pi-lgwjj        0/1     Completed   0          25m
[root@k8s-master home]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     1/1           64s        28m
```

查看job执行结果，如下：

```
[root@k8s-master home]# kubectl logs pi-lgwjj
3.141592653589793238462......
```

## 2.创建定时任务

生产创建JOB的yaml文件cronjob-test.yaml，内容如下：

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello suxianwei from the Kubernetes cluster
          restartPolicy: OnFailure
```

执行创建命令，并查看（每隔1分钟执行一次，每次执行就是生成一个job并执行）：

```
[root@k8s-master home]# kubectl apply -f cronjob-test.yaml 
cronjob.batch/hello created
[root@k8s-master home]# kubectl get pod
NAME                     READY   STATUS              RESTARTS   AGE
hello-1618736640-db9wx   0/1     Completed           0          2m10s
hello-1618736700-hksds   0/1     Completed           0          70s
hello-1618736760-lxf5t   0/1     ContainerCreating   0          10s
[root@k8s-master home]# kubectl get job
NAME               COMPLETIONS   DURATION   AGE
hello-1618736640   1/1           17s        2m29s
hello-1618736700   1/1           17s        89s
hello-1618736760   1/1           17s        29s
[root@k8s-master home]# kubectl get cronjob
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     1        10s             28s
```

查看job执行结果，如下：

```
[root@k8s-master home]# kubectl logs hello-1618735920-k6g28
Sun Apr 18 08:52:19 UTC 2021
Hello suxianwei from the Kubernetes cluster
[root@k8s-master home]# kubectl logs hello-1618736220-8q7z2
Sun Apr 18 08:57:20 UTC 2021
Hello suxianwei from the Kubernetes cluste
```

