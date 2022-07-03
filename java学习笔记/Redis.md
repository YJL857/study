# Redis

### docker搭建redis集群

3主3从

```shell
docker pull redis:6.0.8

docker run -d --name redis-node-1 --net host --privileged=true -v /data/redis/share/redis-node-1:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6381

docker run -d --name redis-node-2 --net host --privileged=true -v /data/redis/share/redis-node-2:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6382

docker run -d --name redis-node-3 --net host --privileged=true -v /data/redis/share/redis-node-3:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6383

docker run -d --name redis-node-4 --net host --privileged=true -v /data/redis/share/redis-node-4:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6384

docker run -d --name redis-node-5 --net host --privileged=true -v /data/redis/share/redis-node-5:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6385

docker run -d --name redis-node-6 --net host --privileged=true -v /data/redis/share/redis-node-6:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6386

一共复制6个
```

命令详解：

--net host：使用宿主机的IP和端口

--privileged=true：获取宿主机root用户权限

--cluster-enabled yes：开启redis集群

--appendonly yes：开启持久化

--port 6381：redis端口号

### 进入其中一个容器

```shell
docker exec -it dd5b417aad14 /bin/bash
```

### 构建主从关系

进入docker容器后执行下面的命令，注意是真实IP地址

```shell
redis-cli --cluster create 43.129.231.24:6381 43.129.231.24:6382 43.129.231.24:6383 43.129.231.24:6384 43.129.231.24:6385 43.129.231.24:6386 --cluster-replicas 1
```

命令详解：

--cluster-replicas 1 为每一个master创建一个slave节点

运行结果：哈希槽分配图

![image-20220702230718481](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220702230718481.png)

### 以6381为切入点，查看集群状态

```shell
redis-cli -p 6381

cluster info

cluster nodes
```

结果：

![image-20220702232051964](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220702232051964.png)

主从关系图

Mster  slave

6381   6385

6382   6386

6383   6384

###  

单机连接redis(在集群中，有的key不在哈希槽范围内，会报错)

```shell
redis-cli -p 6381
```

集群连接redis（防止路由失效，加参数  c）

```
redis-cli -p 6381 -c
```

查看所有的key

```
keys *
```



### 查看集群信息

```shell
redis-cli --cluster check 43.129.231.24:6381
```

![image-20220702233943352](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220702233943352.png)

### 主从容错切换迁移

#### Master宕机，slave会不会上位？

```shell
# 停掉6381节点
docker stop redis-node-1
# 进入6382容器
docker exec -it redis-node-2 /bin/bash
# 以6382为切入点
redis-cli -p 6382 -c
# 查看集群节点
cluster nodes
```

![image-20220702234832504](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220702234832504.png)

由上图可以看到：

  6381节点已经宕机了，6385作为slave节点，已经上位了。

#### 如果6381恢复了，他还是Master吗？

```shell
# 启动6381节点
docker start redis-node-1
# 进入6381容器
docker exec -it redis-node-1 /bin/bash
# 以6381为切入点
redis-cli -p 6381 -c
# 查看集群节点
cluster nodes
```

结果：

![image-20220702235553060](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220702235553060.png)

6381还是slave，6385还是master。曾经的Master不会变回来了！

### redis集群扩容

##### 新建6387，6388两个节点

```shell
docker run -d --name redis-node-7 --net host --privileged=true -v /data/redis/share/redis-node-7:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6387

docker run -d --name redis-node-8 --net host --privileged=true -v /data/redis/share/redis-node-8:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6388
```

##### 进入6387容器实例内部

```shell
docker exec -it redis-node-7 /bin/bash
```

##### 将新增的6387节点（空槽号）作为master节点加入原集群

```shell
redis-cli --cluster add-node 43.129.231.24:6387 43.129.231.24:6381
```

6381就是原集群节点里面的领路人，相当于6387拜托6381加入集群

![image-20220703003637116](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220703003637116.png)

##### 检查集群

```shell
redis-cli --cluster check 43.129.231.24:6381
```

![image-20220703004447769](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220703004447769.png)

6387已经加入集群了，但是没有分配槽位



##### 重新分配槽号

```shell
redis-cli --cluster reshard 43.129.231.24:6381
```

![image-20220703005002728](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220703005002728.png)

4太主机均分  ：  16384/4=4096 

![image-20220703005200307](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220703005200307.png)

给6387分

后面写

```shell
all
yes
```

##### 在检查一下集群

```shell
redis-cli --cluster check 43.129.231.24:6381
```

![image-20220703005443828](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220703005443828.png)

说明不是全部重新洗牌，而是从原先的集群的槽中，各自节点匀一点槽号出来给新的节点。保证数据不是发生变化。



##### 为6387分配从节点

```
redis-cli --cluster add-node 43.129.231.24:6388 43.129.231.24:6387 --cluster-slave --cluster-master-id f1163136809941af595786f6dc3a4ef2db3f7ea1
```

![image-20220703010111554](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220703010111554.png)

Master  slave

6387       6388

### redis集群缩容

##### 先删除slave（从）节点，6388

```shell
redis-cli --cluster del-node 43.129.231.24:6388 eac43ee204d709ec48b816f989e7007c106d0839
```

![image-20220703011107577](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220703011107577.png)

##### 将6387（Master）的槽号清空，重新分配，将槽号都给6381

```shell
redis-cli --cluster reshard 43.129.231.24:6381
```

![image-20220703012105926](C:\Users\FirstDream\AppData\Roaming\Typora\typora-user-images\image-20220703012105926.png)

##### 再查询一次集群

```shell
redis-cli --cluster check 43.129.231.24:6381
```

##### 将6387从集群中删除（ip，端口，节点id）

```bash
redis-cli --cluster del-node 43.129.231.24:6387 f1163136809941af595786f6dc3a4ef2db3f7ea1
```



### redis事物(集群不支持事务)

- 开启事物

```bash
multi
```

- 输入命令

- 执行事务

```bash
exec
```

> 放弃事物
