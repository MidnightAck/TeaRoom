# Ubuntu 18.04 安装docker

环境：虚拟机Ubuntu18.04,[【Ubuntu18.04】安装Docker教程\_运维\_iefenghao的博客-CSDN博客](https://blog.csdn.net/iefenghao/article/details/90747642)

首先更新源

```text
sudo apt update
```

安装依赖

```text
sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

添加Dokcer官方密钥到系统中

```text
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

添加Docker源

```text
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
```

更新一下源

```text
sudo apt update
```

查看可以安装的docker版本

```text
apt-cache policy docker-ce
```

开始安装docker

```text
sudo apt install docker-ce
```

测试

```text
docker --version
```

```text
sudo docker run hello-world
```

以上安装Docker成功

