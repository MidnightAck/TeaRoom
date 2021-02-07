# Ubuntu18.04 通过docker安装ceph集群

这里的是网上直接找的，亲测可以实现，但一定要注意换源！下面代码里换的是cn的源，我自己用是不行的，换成aliyun的源就可以了。这里可以看这篇文章\([修改docker镜像源的方法\_运维\_skh2015java的博客-CSDN博客](https://blog.csdn.net/skh2015java/article/details/82631633)

```text
# 要用root用户创建，或有sudo权限
# 注：docker镜像源(镜像加速)：https://registry.docker-cn.com
# 1、修改docker镜像源
cat > /etc/docker/deamon.json << EOF
{
    "registry-mirrors": [
        "https://registry.docker-cn.com"
    ]
}
EOF

# 需要用到的镜像：ceph/mon, ceph/osd, ceph/radosgw
# 重启docker
systemctl restart docker
# 2、创建ceph专用网络
docker network create --driver bridge --subnet 172.20.0.0/16 ceph-network
docker network inspect ceph-network

# 3、删除旧的ceph相关容器
docker rm -f $(docker ps -a | grep ceph | awk '{print $1}')
# 4、清理旧的ceph相关目录文件，
rm -rf /www/ceph /var/lib/ceph /www/osd
# 5、创建相关目录及修改权限，用于挂载volume
mkdir -p /www/ceph /var/lib/ceph/osd /www/osd/
chown -R 64045:64045 /var/lib/ceph/osd
chown -R 64045:64045 /www/osd/

# 6、创建monitor节点
docker run -itd --name monnode --network ceph-network --ip 172.20.0.10 -e NON_NAME=monnode -e MON_IP=172.20.0.10 -v /www/ceph:/etc/ceph ceph/mon
# 7、在monitor节点上标识3个osd节点
docker exec monnode ceph osd create
docker exec monnode ceph osd create
docker exec monnode ceph osd create

# 8、创建osd节点
docker run -itd --name osdnode0 --network ceph-network -e CLUSTER=ceph -e WEIGHT=1.0 MON_NAME=monnode -e MON_IP=172.20.0.10 -v /www/ceph:/etc/ceph -v /www/osd0:/var/lib/ceph/osd/ceph-0 ceph/osd

docker run -itd --name osdnode1 --network ceph-network -e CLUSTER=ceph -e WEIGHT=1.0 MON_NAME=monnode -e MON_IP=172.20.0.10 -v /www/ceph:/etc/ceph -v /www/osd1:/var/lib/ceph/osd/ceph-1 ceph/osd

docker run -itd --name osdnode2 --network ceph-network -e CLUSTER=ceph -e WEIGHT=1.0 MON_NAME=monnode -e MON_IP=172.20.0.10 -v /www/ceph:/etc/ceph -v /www/osd2:/var/lib/ceph/osd/ceph-2 ceph/osd

# 9、增加monitor节点，组件成机器
docker run -itd --name monnode_1 --network ceph-network --ip 172.20.0.11 -e NON_NAME=monnode_1 -e MON_IP=172.20.0.11 -v /www/ceph:/etc/ceph ceph/mon
docker run -itd --name monnode_2 --network ceph-network --ip 172.20.0.12 -e NON_NAME=monnode_2 -e MON_IP=172.20.0.12 -v /www/ceph:/etc/ceph ceph/mon

# 10、创建gateway节点
docker run -itd --name gwnode --network ceph-network --ip 172.20.0.9 -p 9080:80 -e RGW_NAME=gwnode -v /www/ceph:/etc/ceph ceph/radoosgw

# 11、查看ceph集群状态
sleep 10 && docker exec monnode ceph -s
```

