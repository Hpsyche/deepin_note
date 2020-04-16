# 前言:

在分布式架构中，往往会有多个tomcat，然后你上传的图片只是在其中的某一个tomcat，你访问时是由集群的tomcat随机提供服务。当你访问的tomcat是有图片的那个时，图片能正常显示，如果恰巧是那个没有图片的tomcat时，图片就不能正常显示。这就完成了访问同一个图片，可能你刷新一次可以访问，再刷新一次图片就访问不到了。这时，我们就需要一个服务器用来专门存储图片，一般我们都用nginx。

# 简介:

## 1、nginx:

Nginx是一款轻量级的Web 服务器/反向代理服务器及电子邮件（IMAP/POP3）代理服务器，并在一个BSD-like 协议下发行。其特点是占有内存少，并发能力强，事实上nginx的并发能力确实在同类型的网页服务器中表现较好，中国大陆使用nginx网站用户有：百度、京东、新浪、网易、腾讯、淘宝等。以上是百度百科的介绍，我们目前只需要知道nginx是一个服务器就行了，类似于tomcat的服务器，只不过我们把它用来保存图片。

## 2、vsftp:

VSFTP是一个基于GPL发布的类Unix系统上使用的FTP服务器软件，它有安全、高速、稳定等特点。我们暂且这样理解:vsftp就是用来传输文件的一个服务，在linux系统中开启vsftp服务，然后在windows中就可以通过linux系统的ip、vsftp服务的端口、vsftp的用户名及密码连接vsftp服务，然后就可以方便的把windows中东西上传到linux中，也可以把linux中的东西下载到windows中。

## 3、nginx+vsftp:

上面分别介绍了nginx和vsftp，那么这两个东西怎么组合起来用呢？怎么实现这个图片服务器呢？我们知道，tomcat安装好启动后，在浏览器输入`localhost:8080`，就会出现tomcat的欢迎页，nginx也一样。比如linux的ip是`192.168.50.122`，那么启动nginx后，在浏览器访问这个地址也会出现nginx的欢迎页，其实是因为它有个默认的访问页面，完整的地址应该是`192.168.50.122/index.html`，那么我们就可以根据这个，把它默认的访问页面改成我们上传的图片的保存路径，比如上传了一张pic.jpg图片到linux的`/home/ftpuser/images`中，如果我们把默认访问页面改成`/home/ftpuser`，那么在浏览器中输入`192.168.50.122/images/pic.jpg`，就可以访问到这张图片了。下面就来介绍nginx、vsftp的安装以及配置。

# nginx的安装:

## 1、环境:

nginx是C语言开发，建议在linux上运行，本教程使用Centos 7作为安装环境。先要安装如下东西:
 **①、gcc:**



```swift
yum install gcc-c++ 
```

**②、pcre:**



```undefined
yum install -y pcre pcre-devel
```

**③、zlib:**



```undefined
yum install -y zlib zlib-devel
```

**④、openssl:**



```undefined
yum install -y openssl openssl-devel
```

**⑤、开启防火墙端口:**
 我们把nginx和vsftp要用到的端口先开启，免得后面出错:



```csharp
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent
firewall-cmd --zone=public --add-port=22/tcp --permanent
firewall-cmd --zone=public --add-port=21/tcp --permanent
firewall-cmd --zone=public --add-port=30000-30999/tcp --permanent
```

将以上5条命令逐一执行就行了。
 完成以上安装和设置，就可以开始安装nginx了。

## 2、安装nginx:

**①、下载:**



```swift
wget -c https://nginx.org/download/nginx-1.10.1.tar.gz
```

版本大家可以上官网看一下，把版本号改成自己想下的那个就行了。

**②、解压:**



```css
tar -zxvf nginx-1.10.1.tar.gz
cd nginx-1.10.1
```

**③、设置编译参数:**



```ruby
./configure \
--prefix=/usr/local/nginx \
--pid-path=/var/run/nginx/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--http-scgi-temp-path=/var/temp/nginx/scgi
```

直接把这段代码贴到linux中执行就行了。

**④、编译:**



```go
make
```

**⑤、安装:**



```go
make install
```

**⑥、启动nginx:**



```bash
cd /usr/local/nginx/sbin
./nginx
```

执行这个命令后是没有任何提示的，然后在浏览器中访问虚拟机的ip，出现nginx欢迎页则安装成功。

**⑦、关闭nginx:**
 在刚才的sbin目录下执行:

```undefined
./nginx -s stop
```

**遇到的坑:**
 第一次启动nginx没问题，如果重启了一下虚拟机，再次到nginx的sbin目录下执行`./nginx`，出现下图所示的错误:

![img](https:////upload-images.jianshu.io/upload_images/11531502-4460bb8f9c5ec56b.png?imageMogr2/auto-orient/strip|imageView2/2/w/895/format/webp)



**解决办法:**
 在run文件夹下创建一个nginx文件夹即可。

```bash
cd /var/run
mkdir nginx
```

创建nginx文件夹后成功启动:

![img](https:////upload-images.jianshu.io/upload_images/11531502-da2e7318920a2e41.png?imageMogr2/auto-orient/strip|imageView2/2/w/821/format/webp)

但是我发现每次重启了虚拟机这个nginx文件夹都会被干掉，每次都要重新创建nginx文件夹才能启动nginx，不知道是何原因。知道的老铁们请赐教哦！

# vsftp的安装:

**1、安装:**

```undefined
yum -y install vsftpd
```

**2、添加ftp用户:**

```undefined
useradd ftpuser
```

**3、给ftp用户添加密码:**

```undefined
passwd ftpuser
```

输入两次密码后修改密码。

**4、修改selinux:**
 **①查看状态:**

```undefined
getsebool -a | grep ftp
```

执行这个命令可以看到

```rust
allow_ftpd_full_access --> off
ftp_home_dir --> off
```

这两个都off，执行如下命令设置为on:

```csharp
[root@localhost ~]# setsebool -P ftpd_full_access on
[root@localhost ~]# setsebool -P ftp_home_dir on
```

再次执行`getsebool -a | grep ftp`看到那两个状态是on就行了。

**5、关闭匿名访问:**
 执行

```undefined
vim /etc/vsftpd/vsftpd.conf
```

命令:

将anonymous_enable改为NO

![img](https:////upload-images.jianshu.io/upload_images/11531502-8cc33c340eb27366.png?imageMogr2/auto-orient/strip|imageView2/2/w/794/format/webp)

还要在vsftp.conf文件最下面添加以下内容:

![img](https:////upload-images.jianshu.io/upload_images/11531502-bb5e6e2cbabdf6f4.png?imageMogr2/auto-orient/strip|imageView2/2/w/592/format/webp)

pasv_min_port=30000

pasv_max_port=30999

 然后保存退出即可。

**6、设置开机启动:**

```csharp
[root@localhost ~]# chkconfig vsftpd on
```

**7、测试:**
 打开filezilla工具，输入虚拟机的ip，21端口，用户名和密码，点击快速连接，连接vsftp服务:

![img](https:////upload-images.jianshu.io/upload_images/11531502-6822920669761223.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/700/format/webp)

如图所示则连接成功。

# 配置nginx为图片服务器:

**按照以上步骤安装好nginx和vsftp后，还是不能访问上传的图片的，需要进行如下配置:**
 执行

```bash
vim  /usr/local/nginx/conf nginx.conf
```

命令，打开nginx的配置文件:

![img](https:////upload-images.jianshu.io/upload_images/11531502-8551b471c0c0b209.png?imageMogr2/auto-orient/strip|imageView2/2/w/628/format/webp)

将server中的location中的root改为/home/ftpuser

按道理这样就可以了，但是我访问却报错:
 **403 forbidden**，最后发现是因为ftpuser文件夹没有可读权限，执行如下命令:

```undefined
chmod -R 755 /home/ftpuser
```

再次访问即可成功！

![img](https:////upload-images.jianshu.io/upload_images/11531502-4c2eeeb3529a76de.png?imageMogr2/auto-orient/strip|imageView2/2/w/503/format/webp)

![img](https:////upload-images.jianshu.io/upload_images/11531502-15310401d3db105d.png?imageMogr2/auto-orient/strip|imageView2/2/w/943/format/webp)

至此图片服务器搭建完成！