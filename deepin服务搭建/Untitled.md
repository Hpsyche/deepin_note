## 1. 搭建mysql

```
docker run -p 3306:3306 --name docker-mysql-5.7 -v $PWD/conf:/etc/mysql/conf.d -v $PWD/logs:/logs -v $PWD/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
```

## 2. docker logs 容器id

2019-08-03T10:03:57.601772Z 0 [ERROR] Failed to create a socket for IPv4 '0.0.0.0': errno: 13.
2019-08-03T10:03:57.601779Z 0 [ERROR] Can't create IP socket: Permission denied
2019-08-03T10:03:57.601782Z 0 [ERROR] Aborting

## 3. 过几秒后，容器退出

# 解决方法

```
sudo apt remove apparmor
```

### 切换用户需要source /etc/profile

也可以放在~/.bashrc里面。或者在~/.bashrc里面加一句source /etc/profile