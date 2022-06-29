## 获取镜像

docker pull [选项] [Docker Registry 地址[:端口]/]仓库名[:标签]

example:
docker pull ubuntu:16.04

docker run -it --rm \
    ubuntu:16.04 \
    /bin/bash
:-i 表示交互式操作 -t 表示终端
:--rm 表示容器退出后随之将其删除
:-d 表示后台运行
## 列出镜像
docker image ls

docker system df：查看镜像、容器、数据卷所占用的空间。
## 启动已经终止的容器
docker container start

## 终止容器
docker container stop

docker container restart

## 进入后台启动的容器
docker exec -it CONTAINER ID bash
## 删除处于终止状态的容器(要删除运行中的容器要加参数-f)
docker container rm CONTAINER NAME

## 查看已创建的包括终止黄台的容器
docker ps -a 
## 清理所有处于终止状态的容器
docker container prune

## 删除本地镜像
docker image rm[选项]<镜像1>[<镜像2>...]

## 查看本地镜像
docker image ls

## 查看镜像内历史记录
docker history


## 数据共享与持久化
(1)Data Volumes:
可以在容器之间共享和重用
对数据卷的修改会立马生效
对数据卷的更新,不会影响镜像
数据卷会默认一直存在,即使容器被删除
### Volume 相关操作
创建数据卷：
```docker
$ docker volume create my-vol
```
查看所有的数据卷：
```docker
$ docker volume ls
local               my-vol
```
查看指定数据卷的信息
```docker 
$ docker volume inspect my-vol
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```
删除数据卷：
```docker
$ docker volume rm my-vol
```
若需要删除容器时同时移除数据卷
```docker
docker rm -v
```

清理无主数据卷：
```docker
docker volume prune
```

(2)Bind mounts
