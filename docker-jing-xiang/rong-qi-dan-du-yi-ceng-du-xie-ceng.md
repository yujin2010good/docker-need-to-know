---
description: 容器单独一层读写层，容器只读镜像
---

# 容器是单独一层读写层

实际中，我们发现一个镜像可以run无数个容器，容器需要读取文件的场景和对应原理是如下\(这里先将这个规则，后续也会去验证下面这些规则\)

1. \(无数据卷和挂载下\)向容器添加文件（docker cp进去或者exec在里面下载或者命令产生文件） 文件在容器这层读写层里,容器删除会则删除
2. 读取文件 从上层往下找到镜像层,找到即可返回,复制到容器层读入内存
3. 修改文件 从上层往下找到镜像层,找到即可返回，复制到容器层后修改
4. 删除文件 找到后在容器层记录下删除操作\(后续读取的时候会认为文件不存在\)

容器与镜像关系为下图，实际上还有层init层，后面讲

![](../.gitbook/assets/image%20%2869%29.png)

通过docker ps 的-s选项可以看出容器的size和容器层总大小，这里我用docker命令演示下容器是单独一层读写层和容器死亡数据丢失\(也就是说容器是无状态的\)

![](../.gitbook/assets/image%20%2834%29.png)

创建一个容器，在容器里写入1g数据，宿主机的可用容量减少1G，docker的overlay2存储目录记录了下这个文件，但是删除后文件也被删除了。在抽象逻辑上一个容器就是单独一个读写层，而删除容器后这层在宿主机上的落地文件也会被删除。

计算实际占用大小是镜像的大小不会被重复计算，只需要计算一个大小+它起的所有容器大小。容器是只读镜像

假设起了五个nginx

```text
$ docker ps -s
CONTAINER ID        IMAGE               COMMAND                  CREATED                  STATUS              PORTS     SIZE
8d73ab60d109        nginx:alpine        "nginx -g 'daemon of…"   Less than a second ago   Up About a minute   80/tcp    2B (virtual 15.5MB)
3020c51822ce        nginx:alpine        "nginx -g 'daemon of…"   Less than a second ago   Up About a minute   80/tcp    2B (virtual 15.5MB)
189e575ab705        nginx:alpine        "nginx -g 'daemon of…"   Less than a second ago   Up About a minute   80/tcp    2B (virtual 15.5MB)
ac1146052c53        nginx:alpine        "nginx -g 'daemon of…"   Less than a second ago   Up About a minute   80/tcp    2B (virtual 15.5MB)
f4a5c9b7bef3        nginx:alpine        "nginx -g 'daemon of…"   Less than a second ago   Up About a minute   80/tcp    2B (virtual 15.5MB)
$ docker image ls nginx:alpine
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               alpine              21a2450d623e        8 weeks ago         15.5MB
```

然后使用exec在容器里写数据

```text
docker exec web1 sh -c 'dd if=/dev/zero of=/test.log bs=1000000 count=10'
docker exec web2 sh -c 'dd if=/dev/zero of=/test.log bs=1000000 count=20'
docker exec web3 sh -c 'dd if=/dev/zero of=/test.log bs=1000000 count=30'
docker exec web4 sh -c 'dd if=/dev/zero of=/test.log bs=1000000 count=40'
docker exec web5 sh -c 'dd if=/dev/zero of=/test.log bs=1000000 count=50'
$ docker ps -as
CONTAINER ID        IMAGE               COMMAND                  CREATED                  STATUS             PORTS     SIZE
83ddc6c59c8b        nginx:alpine        "nginx -g 'daemon of…"   2 minutes ago            Up 2 minutes       80/tcp    50MB (virtual 65.5MB)
037318ee3cc6        nginx:alpine        "nginx -g 'daemon of…"   2 minutes ago            Up 2 minutes       80/tcp    40MB (virtual 55.5MB)
cebf853b65ca        nginx:alpine        "nginx -g 'daemon of…"   2 minutes ago            Up 2 minutes       80/tcp    30MB (virtual 45.5MB)
fdf1998052d1        nginx:alpine        "nginx -g 'daemon of…"   2 minutes ago            Up 2 minutes       80/tcp    20MB (virtual 35.5MB)
922c23c0e38a        nginx:alpine        "nginx -g 'daemon of…"   2 minutes ago            Up 2 minutes       80/tcp    10MB (virtual 25.5MB)
```

实际占用计算为

```text
1 x 15.5MB 只读的镜像层
1 x 10MB
1 x 20MB
1 x 30MB
1 x 40MB
1 x 50MB
===========================+
165.5MB
```

这样我们可以逆推出docker镜像是分层和容器是单独一层只读镜像的。也有部分人不懂这些知识，每次是进容器里安装东西然后commit，导致最后容器越来越大，甚至看到过16g的镜像

