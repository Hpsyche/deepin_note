### 一下载依赖包

1.下载Rabbitmq 所依赖的环境gcc、erlang包和rabbitmq包，这里演示是网上下载
gcc

依赖

```
yum install build-essential openssl openssl-devel unixODBC unixODBC-devel make gcc gcc-c++ kernel-devel m4 ncurses-devel tk tc xz
```

erlang依赖

```
wget www.rabbitmq.com/releases/erlang/erlang-``18.3``-``1``.el7.centos.x86_64.rpm
```

rabbitmq包

```
wget www.rabbitmq.com/releases/rabbitmq-server/v3.``6.5``/rabbitmq-server-``3.6``.``5``-``1``.noarch.rpm
```

　　

###  二.使用rpm -ivh 命令安装erlang包和rabbitmq包

```
rpm -ivh erlang-``18.3``-``1``.el7.centos.x86_64.rpm
rpm -ivh rabbitmq-server-``3.6``.``5``-``1``.noarch.rpm
```

注意*安装rabbitmq这个包时候提示错误缺少一个socat依赖
我们用yum把它装上

```
yum install socat
```

装完后再

rpm -ivh rabbitmq-server-3.6.5-1.noarch.rpm

### *三.修改host和hostname*

```
vim /etc/hostname
```

　　添加个yux

```
vim /etc/hosts
```

　　192.168.6.42 yux #主机 加上你的主机名（装Rabbitmq的主机

### *四.给rabbitmq加入密码*

```
vim /usr/lib/rabbitmq/lib/rabbitmq_server-``3.6``.``5``/ebin/rabbit.app
```

　　修改 {loopback_users, [guest]},

五.运行rabbitmq 

```
rabbitmq-server start &
```

　6.查看rabbitmq插件

```
rabbitmq-plugins list
```

7.开启客户端管理连接



```
rabbitmq-plugins enable rabbitmq_management
```

　　

8.浏览器访问
http://192.168.6.42:15672/
guest/guest

![img](https://img2018.cnblogs.com/blog/1476521/201809/1476521-20180919173600473-335000560.png)