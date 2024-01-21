---
title: Docker
categories:
  - Container
tags:
  - Docker
abbrlink: c018
date: 2024-01-21 22:49:15
---
### Docker Engine
#### Install
```shell
# install docker engine
https://docs.docker.com/engine/install/debian/

```

#### Storage
##### Overview
```shell
# show docker volume info
docker volume ls
DRIVER    VOLUME NAME
local     jenkins_home
local     yakir-test


# how to use
# default volume, directory = /var/lib/docker/volumes/
-v yakir-test:/container-app/my-app
--volume yakir-test:/container-app/my-app
--mount
# bind mounts
-v /local_path/app.conf:/container-app/app.conf
--volume /local_path/app.conf:/container-app/app.conf
--mount
# memory volume
--tmpfs

```
<!--more-->
##### Volumes
```shell
# create volume
docker volume create yakir-test


# start container with volume
docker run -d --name test \
### 
# option1
-v yakir-test:/app \
--volume yakir-test:/app \
# anonymous mode
--volume /app
# option2
--mount source=yakir-test,target=/app \
# readonly mode
--mount source=yakir-test,destination=/usr/share/nginx/html,readonly \
--mount 'type=volume,source=nfsvolume,target=/app,volume-driver=local,volume-opt=type=nfs,volume-opt=device=:/var/docker-nfs,volume-opt=o=addr=10.0.0.10' \
###
nginx:latest


# use a volume with docker-ompose
services:
  frontend:
    image: node:lts
    volumes:
      - yakir-test:/home/node/app
volumes:
  yakir-test:
     # external: true


# show and remove volume
docker inspect volume yakir-test
docker stop test
docker volume rm yakir-test

```

##### Bind mounts
```shell
# start container with bind mounts
docker run -d --name test \
###
# option1
-v /opt/app.conf:/app/app.conf \
# option2
--mount type=bind,source="$(pwd)"/target,target=/app/ \
--mount type=bind,source="$(pwd)"/target,target=/app/,readonly \
# bind propagation
--mount type=bind,source="$(pwd)"/target,target=/app2,readonly,bind-propagation=rslave \
###
nginx:latest


# use bind mounts with docker-compose
services:
  frontend:
    image: node:lts
    volumes:
      - type: bind
        source: ./static
        target: /opt/app/static
volumes:
  myapp:


# show and remove container
docker inspect test --format '{{ json .Mounts }}'
docker stop test
docker rm test

```

##### tmpfs mounts
```shell
# start container with tmpfs
docker run -it --name tmptest \
###
# option1
--tmpfs /app
# option2
--mount type=tmpfs,target=/app \
# specify tmpfs options
--mount type=tmpfs,destination=/app,tmpfs-mode=1770,tmpfs-size=104857600 \
###
nginx:latest


# show and remove container
docker inspect tmptest --format '{{ json .Mounts }}'
docker stop tmptest
docker rm tmptest

```

##### Storage drivers
###### Btrfs
```shell
# stop docker
systemctl stop docker.service

# backup and empty contents
cp -au /var/lib/docker/ /var/lib/docker.bk
rm -rf /var/lib/docker/*

# format block device as a btrfs filesystem
mkfs.btrfs -f /dev/xvdf

# mount the btrfs filesystem on /var/lib/docker mount point
mount -t btrfs /dev/xvdf /var/lib/docker
cp -au /var/lib/docker.bk/* /var/lib/docker/

# configure Docker to use the btrfs storage driver
vim /etc/docker/daemon.json
{
  "storage-driver": "btrfs"
}
systemctl start docker.service

# verify
docker info --format '{{ json .Driver }}'
"btrfs"
```

###### OverlayFS
```shell
# stop docker
systemctl stop docker.service

# backup and empty contents
cp -au /var/lib/docker/ /var/lib/docker.bk
rm -rf /var/lib/docker/*

# options: separate backing filesystem, mount into /var/lib/docker and make sure to add mount to /etc/fstab to make it.  

# configure Docker to use the btrfs storage driver
vim /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}
systemctl start docker.service

# verify
docker info --format '{{ json .Driver }}'    
"overlay2"
mount |grep overlay |grep docker
```

###### ZFS
```shell
# stop docker
systemctl stop docker.service

# backup and empty contents
cp -auR /var/lib/docker/ /var/lib/docker.bk
rm -rf /var/lib/docker/*

# create a new zpool on block device and mount into /var/lib/docker
zpool create -f zpool-docker -m /var/lib/docker /dev/xvdf
# add zpoll
zpool add zpool-docker /dev/xvdh
# verify zpool
zfs list
NAME           USED  AVAIL  REFER  MOUNTPOINT
zpool-docker    55K  96.4G    19K  /var/lib/docker

# configure Docker to use the btrfs storage driver
vim /etc/docker/daemon.json
{
  "storage-driver": "zfs"
}
systemctl start docker.service

# verify
docker info --format '{{ json .Driver }}'    
"zfs"
```

###### containerd snapshotters
```shell
# configure Docker to use the btrfs storage driver
vim /etc/docker/daemon.json
{
  "features": {
    "containerd-snapshotter": true
  }
}
systemctl restart docker.service

# verify
docker info -f '{{ .DriverStatus }}'
[[driver-type io.containerd.snapshotter.v1]]

```

#### Networking
##### Overview

```shell
# show docker network info
docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
b2adc1fcf214   bridge    bridge    local
2ed9fbc8db3e   host      host      local
f1b2d749ed2c   none      null      local

# how to use
# bridge
--net bridge
# host
--net host
# none
--net none
# container
--net container:container_name|container_id

```

##### Networking drivers
###### Bridge
```shell
# bridge
每个容器拥有独立网络协议栈，为每一个容器分配、设置 IP 等。将容器连接到虚拟网桥（默认为 docker0 网桥）。

# 1.在宿主机上创建 container namespace
xxx

# 2.daemon 进程利用 veth pair 技术，在宿主机上创建一对对等虚拟网络接口设备。veth pair 特性是一端流量会流向另一端。
# 一个接口放在宿主机的 docker0 虚拟网桥上并命名为 vethxxx
# 查看网桥信息
brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242db01d347       no              vethccab668
# 查看宿主机 vethxxx 接口
ip addr |grep vethccab668
# 另外一个接口放进 container 所属的 namespace 下并命名为 eth0 接口
docker run --rm -dit busybox sh ip addr

# 3.daemon 进程还会从网桥 docker0 的私有地址空间中分配一个 IP 地址和子网给该容器，并设置 docker0 的 IP 地址为容器的默认网关
docker inspect test |grep Gateway
            "Gateway": "172.17.0.1",

```

###### Overlay
```shell
# 多 docker 主机组建网络，配合 docker swarm 使用
```

###### Host
```shell
# host
使用宿主机的 IP 和端口，共享宿主机网络协议栈。

# test
docker run --rm -dit --net host busybox ip addr
```

###### IPvlan
```shell
# ipvlan
ipvlan_mode: l2, l3(default), l3s
ipvlan_flag: bridge(default), private, vepa
parent: eth0

# l2 mode: 使用宿主机的望断
docker network create -d ipvlan \
     --subnet=192.168.1.0/24 \
     --gateway=192.168.1.1 \
     -o ipvlan_mode=l2 \
     -o parent=eth0 test_l2_net
# test
docker run --net=test_l2_net --name=ipv1 -dit alpine /bin/sh
docker run --net=test_l2_net --name=ipv2 -it --rm alpine /bin/sh
ping -c 4 ipv1

# l3 mode
docker network create -d ipvlan \
     --subnet=192.168.1.0/24 \
     --subnet=10.10.1.0/24 \
     -o ipvlan_mode=l3 test_l3_net
# test
docker run --net=test_l3_net --ip=192.168.1.10 -dit busybox /bin/sh
docker run --net=test_l3_net --ip=10.10.1.10 -dit busybox /bin/sh

docker run --net=test_l3_net --ip=192.168.1.9 -it --rm busybox ping -c 2 10.10.1.10
docker run --net=test_l3_net --ip=10.10.1.9 -it --rm busybox ping -c 2 192.168.1.10

```

###### Macvlan
```shell
# macvlan

# bridge mode
docker network create -d macvlan \
  --subnet=172.16.86.0/24 \
  --gateway=172.16.86.1 \
  -o parent=eth0 pub_net


# 802.1Q trunk bridge mode
docker network create -d macvlan \
    --subnet=192.168.50.0/24 \
    --gateway=192.168.50.1 \
    -o parent=eth0.50 macvlan50

docker network create -d macvlan \
    --subnet=192.168.60.0/24 \
    --gateway=192.168.60.1 \
    -o parent=eth0.60 macvlan60

# https://zhuanlan.zhihu.com/p/616504632
```

###### None
```shell
# none
每个容器拥有独立网络协议栈，但没有网络设置，如分配 veth pair 和网桥连接等。

# verify
docker run --rm -dit --net none busybox ip addr
```

###### Container
```shell
# container
和一个指定已有的容器共享网络协议栈，使用共有的 IP、端口等。

# verify
docker run -dit --name test --rm busybox sh
docker run -it --name c1 --net container:test --rm busybox ip addr
docker run -it --name c2 --net container:test --rm busybox ip addr
```

###### 自定义网络模式
```shell
# user-defined 
默认 docker0 网桥无法通过 container name host 通信，自定义网络默认使用 daemon 进程内嵌的 DNS server，可以直接通过 --name 指定的 container name 进行通信

# 创建自定义网络
docker network create yakir-test
# 宿主机查看新增虚拟网卡
ip addr
    inet 172.19.0.1/16 brd 172.19.255.255 scope global br-8cb8260a95cf
brctl show
br-8cb8260a95cf         8000.024272aa9d38       no              veth556b81b
# verify
docker run -dit --name test1 --net yakir-test --rm busybox sh
docker run -it --name test2 --net yakir-test --rm busybox ping -c 4 test1

# 连接已有的网络
docker run -dit --name test3 --net yakir-test --rm busybox sh
docker network connect yakir-test test3 
docker exec -it test3 ip addr
531: eth0@if532: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
533: eth1@if534: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:13:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.2/16 brd 172.19.255.255 scope global eth1
       valid_lft forever preferred_lft forever

```

##### Daemon
```shell
# configuration file
/etc/docker/daemon.json
~/.config/docker/daemon.json
# configuration using flags
dockerd --debug \
  --tls=true \
  --tlscert=/var/docker/server.pem \
  --tlskey=/var/docker/serverkey.pem \
  --host tcp://192.168.10.1:2376


# default data directory
/var/lib/docker


# systemd
cat /lib/systemd/system/docker.service


```

#### Docker Build && Compose
##### Dockerfile
```shell

# 编写规范
1. 使用统一的 base 镜像。
2. 动静分离（基础稳定内容放在底层）。
3. 最小原则（镜像只打包必需的东西）。
4. 一个原则（每个镜像只有一个功能，交互通过网络，模块化管理）。
5. 使用更少的层，减少每层的内容。
6. 不要在 Dockerfile 单独修改文件权限（entrypoint / 拷贝+修改权限同时操作）。
7. 利用 cache 加快构建速度。
8. 版本控制和自动构建（放入 git 版本控制中，自动构建镜像，构建参数/变量给予文档说明）。
9. 使用 .dockerignore 文件（排除文件和目录）
```

##### docker-compose
```shell

```

#### Command
[[containerRuntime#docker & podman|Docker Command]]



>Reference:
>1. [Docker Official Documentation](https://docs.docker.com/)
>2. [Docker network-drivers](https://docs.docker.com/network/drivers/)
>3. [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)
