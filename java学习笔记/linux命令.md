# Centos防火墙端口

## 重启docker

systemctl restart docker

## Centos防火墙端口

查看已经开放的端口

firewall-cmd --list-ports

开启防火墙

systemctl start firewalld

关闭防火墙

systemctl stop firewalld;

开启端口

firewall-cmd --zone=public --add-port=6379/tcp --permanent

重启防火墙

firewall-cmd --reload  #重启

firewall systemctl stop firewalld.service  #停止

firewall systemctl disable firewalld.service   #禁制firewall开机启动

# linux高级命令

1. 查看整机性能

   - top

     查看cpu，内存  时间百分比占用

     id（空闲率）

     load average(系统平均负载)，后面有3个参数，代表1分钟，5分钟，15分钟的系统平均负载，这3个值相加除以3，乘以100%，如果大于60%，表示系统负载过重，大于80%表示系统快要崩溃了。

     按键盘上左上角的数字1，打开多核cpu的cpu详情。

     按q退出（别说ctrl C）

   - uptime(简单版)

     查看load average(系统平均负载)

2. 内存

   free

   free -m(显示内存，单位是MB)

   free -g(显示内存，单位是GB)

   free -h(显示内存，-h表示以人类的方式展示，这个命令常用)

3. 硬盘

    df -h(显示硬盘大小)

4. cpu 包含单不限于 vmstat -n 2 3

5. 磁盘IO iostat - xdk 2 3(yum install sysstat)

