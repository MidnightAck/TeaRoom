# Ubuntu 18.04 安装配置redis

环境:Ubuntu18.04 [Ubuntu18.04 安装Redis_数据库_段占彪的博客-CSDN博客](https://blog.csdn.net/github_38336924/article/details/89307739)

### 安装

准备工作

```text
sudo apt update
```

使用 apt 从官方 Ubuntu 存储库来安装 Redis：

```text
sudo apt-get install redis-server
```

### 配置

#### 设置密码

首先打开Redis配置文件redis.conf

```text
sudo vim /etc/redis/redis.conf
```

找到`# requirepass foobared`这一行，将注释符号\#去掉，将后面修改成自己的密码，例如，设置密码为123456

```text
requirepass 123456
```

#### 开启远程访问

默认情况下，Redis服务器不允许远程访问，只允许本机访问，所以我们需要设置打开远程访问的功能。

打开Redis服务器的配置文件`redis.conf`

```text
sudo vi /etc/redis/redis.conf
```

使用注释符号\#注释bind 127.0.0.1这行

```text
#注释bind
#bind 127.0.0.1
```

Redis服务控制

```text
sudo systemctl start redis    #启动
sudo systemctl stop redis    #关闭
sudo systemctl restart redis    #重启
```

注意 修改配置文件之后需要重启Redis服务

