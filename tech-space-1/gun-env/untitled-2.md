# Ubuntu 18.04 通过docker安装Rabbit MQ

环境：Ubuntu 18.04 docker 19.03.8

首先新建个目录，这样将docker容器中的rabbitmq文件也映射一份到本机

```text
sudo mkdir /data/rabbitmq -p
```

接下来让docker去跑rabbitmq，注意下面三个端口，5672是docker映射出来的监听的端口，用于程序访问；15672是用户访问控制台的端口，可以通过浏览器访问的控制台；25672是集群之间的通讯

```text
sudo docker run -d --hostname rabbit-svr --name rabbit -p 5672:5672 -p 15672:15672 -p 25672:25672 -v /data/rabbitmq:/var/lib/rabbitmq rabbitmq:management
```

接下来我们通过浏览器，直接进入15672端口就可以访问控制台。如果rabbitmq在本机，直接是`localhost:15672`,如果是服务器就输入服务器的ip。

控制台的默认用户名和密码都是guest。

