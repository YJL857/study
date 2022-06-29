





1、检查内核版本，必须是3.10及以上

uname -r

虚拟机需要能上外网，ping   www.baidu.com

2、安装docker，

​	1）、安装gcc

​		yum -y install gcc

​	2）、安装gcc+

​		yum -y install gcc-c++

​	3）、安装所需要的软件包

​		yum install -y yum-utils

​	4)、设置stable镜像仓库

​		yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

​	5)、更新yum软件包索引

​		yum makecache fast

​	6)、安装docker-ce引擎

​		yum -y install docker-ce docker-ce-cli containerd.io

3、启动docker

systemctl start docker

​    	停止docker

​		systemctl stop docker

4、查看docker版本

docker -v   或   docker version

5、设置虚拟机开机启动docker

systemctl enable docker

6、设置开机自动重启的容器

docker update 9ad76a7f4949 --restart=always

docker update 10fb6a2d77d2 --restart=no

docker update 10fb6a2d77d2 --restart=on-failure 



Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.



配置阿里云镜像加速器

登录阿里云，控制台，容器镜像，镜像加速器



**docker镜像操作**

项目jar包，制作镜像

1、文件路径

/usr/local/dev

2、编写的`Dockerfile文件为：

FROM openjdk:8-jdk-alpine
VOLUME /tmp
ADD ULServer-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

上面命令的相关解释：

从docker仓库获取openjdk作为我们项目的容器
VOLUME指向了一个/tmp的目录，由于Spring Boot使用内置的Tomcat容器，Tomcat默认使用/tmp作为工作目录。效果就是在主机的/usr/local/dev目录下创建了一个临时文件，并连接到容器的/tmp。
项目的dspring-boot-docker.jar作为app.jar添加到容器。
ENTRYPOINT 执行项目 app.jar。为了缩短 Tomcat 启动时间，添加一个系统属性指向/dev/urandom 作为 Entropy Source

3、构建Docker镜像

在/usr/local/dev目录下，执行`Docker`的命令来构建镜像。

docker build -t yejinliang/ulserver-docker-t:latest .



4、运行容器

docker run -d --name ulserver-docker -p 8081:443 yejinliang/ulserver-docker-t --restart=always
这个表示docker容器在停止或服务器开机之后会自动重新启动 --restart=always 这块可不执行

或者加

docker update --restart=always spring-boot-helloworld-docker

6、docker拉取mysql

​	docker pull mysql

​	docker pull mysql:5.5   //标签

​	列表

​		docker images   //查看本地所有镜像

​	 删除

​		docker rmi image-id  //image-id是镜像id

**docker容器操作**

1、创建一个容器。// --name自定义命名，-d后台运行，tomcat   镜像名

​		docker run --name mytomcat -d tomcat   

2、docker ps //查看运行中的容器

3、停止运行中的容器

​     	docker stop tomcat   //或者写容器的id

​		启动容器

​		docker start tomcat  //或者写容器的id

​		重启容器

​		docker restart 容器id

4、查看所有的容器（运行和不运行）

​		docker ps -a

5、删除容器

​	docker rm  容器id

6、tomcat端口映射

​	 docker run --name mytomcat -d -p 8080:8080 tomcat 

7、docker exec -it 1e33b26152e1 bash    进入容器

​		exit  //退出

8、查看日志

​	docker logs 容器id

### **docker启动mysql**

#### 拉取镜像

```shell
docker pull mysql:5.7
```

#### 运行容器并挂载目录

```shell
docker run -d -p 3307:3306 -v /home/mysql/conf:/etc/mysql/conf.d -v /home/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=111111 --name mysqlmaster mysql:5.7
```

 

#### 容器commit成为一个镜像

docker commit -a="yejinliang" -m="add webapp app" 78c77edb5bb4 tomcat01:1.0

### docker容器中安装vi命令

1、同步源的索引

```sql
apt-get update
```

这个命令的作用是：同步 /etc/apt/sources.list 和 /etc/apt/sources.list.d 中列出的源的索引，这样才能获取到最新的软件包。

2、安装

```csharp
apt-get install vim -y
```

### **通过Docker修改运行中的MySQL容器的时区**

宿主机的时区要是CST，即东八区。
MySQL所在的容器是UTC
在MySQL内执行select now()显示的时间也是UTC。
1. 通过docker cp修改容器时间
如下即可将容器的时间改为和宿主机同样的时区

docker cp /etc/localtime ef058be62c07:/etc/localtime

报错：解决办法：

docker cp /usr/share/zoneinfo/Asia/Shanghai ef058be62c07:/etc/localtime


此时MySQL所在的容器时间如下，已经是东八区

➜  ~ ✗ sudo docker exec -it 231458904a77 /bin/bash         
root@231458904a77:/# date
Thu Jan 28 10:56:42 CST 2021
1
2
3
但是，MySQL服务的时间还不是东八区

2. 重启MySQL容器
sudo docker restart 231458904a77

### 容器数据卷

容器的持久化和同步操作

**使用数据卷**

> 方式一：直接使用命令来挂载 -v

```shell
docker run -it -v 主机目录：容器内目录
```

mysql实战

// 创建临时mysql容器，退出删除

拉取镜像、

```shell
docker pull mysql:5.7
```

```shell
docker run -d -p 3307:3306 -v /home/mysql/conf:/etc/mysql/conf.d -v /home/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=111111 --name mysqlmaster mysql:5.7
```

// 编写配置文件

```shell
vi /home/mysql/conf/my.cnf


my.cnf添加如下内容：
[mysqld]
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

```

// 正式创建mysql容器

```shell
docker run -d -p 3307:3306 -v /home/mysql/conf:/etc/mysql/conf.d -v /home/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=111111 --name mysqlmaster mysql:5.7
```



mysql操作命令（修改权限）

```shell
docker exec -it 容器id

mysql -u root -p

123456

use mysql;
select user, host, plugin from user;
// 创建用户
CREATE USER 'root'@'%' IDENTIFIED BY '123456'; ;
// 修改本地用户的密码
alter user 'root'@'localhost' identified by '123456';

// 分配权限
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
flush privileges;//刷新权限,使操作生效
```

主从同步

// 操作权限控制(master)

// 在Master数据库创建数据同步用户，授予用户 slave REPLICATION SLAVE权限和REPLICATION CLIENT权限，用于在主从库之间同步数据。

CREATE USER 'slave'@'%' IDENTIFIED BY '123456';

GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';

flush privileges;

查看主数据库的状态

show master status;



在Slave 中进入 mysql，执行

```shell
change master to master_host='192.168.31.235', master_user='slave', master_password='123456', master_port=3306, master_log_file='mysql-bin.000001', master_log_pos=156, master_connect_retry=30;
```

在Slave 中的mysql终端执行`show slave status \G;`用于查看主从同步状态

`start slave`开启主从复制过程



```shell

#查看容器信息，包括挂在卷
docker inspect 容器id
```

#### docker 启动redis

 [redis.conf](D:\ChromeCoreDownloads\redis.conf) 文件放入到  /myredis/conf/   下

docker run -p 6379:6379 -v /myredis/conf/redis.conf:/etc/redis/redis.conf -v /myredis/data:/data --name redis01 -d redis:latest redis-server /etc/redis/redis.conf --appendonly yes

#### docker启动rabbitmq

```shell
docker run -d --hostname my-rabbit --name rabbit -p 15672:15672 -p 5673:5672 rabbitmq

docker exec -it 容器id /bin/bssh

rabbitmq-plugins enable rabbitmq_management
```



```
docker run -d --hostname my-rabbit --name rabbit01 -p 5672:15672 -e RABBITMQ_DEFAULT_USER=guest -e RABBITMQ_DEFAULT_PASS=guest -e RABBITMQ_ERLANG_COOKIE='secret cookie here' rabbitmq:3-management
```

