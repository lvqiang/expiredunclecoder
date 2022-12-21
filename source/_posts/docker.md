---
title: docker
date: 2022-12-21 10:06:25
tags:
categoriesa: devops
---



<img src="http://cdn.expiredunclecoder.tech/image-20221201114525791.png" alt="image-20221201114525791" style="zoom:50%;" />

### 镜像相关

#### 查看镜像

```powershell
$ docker images

###如何唯一确定镜像image_id和repository:tag
REPOSITORY    TAG                 IMAGE ID            CREATED             SIZE
nginx         alpine              377c0837328f        2 weeks ago         19.7MB
```

#### 拉取镜像

```bash
$ docker pull nginx:alpine
```

#### 导出导入镜像

```bash
#导出镜像
$ docker save -o nginx-alpine.tar nginx:alpine

#导入镜像
$ docker load -i nginx-alpine.tar
```

#### 删除镜像

```bash
$ docker rmi nginx:alpine

#占用情况
$docker system df 

#批量查看
$ docker images | grep  "image-b" | grep -v "v1d0-7" | awk  '{print $3}'

#批量删除
$ docker rmi $(docker images | grep  "image-b" | grep -v "v1d0-7" | awk  '{print $3}')
```

### 仓库相关

仓库可以选择注册dockerhub、阿里云等第三方镜像仓库，本地搭建镜像仓库可以选择直接使用docker镜像仓库和harbor。

#### 部署镜像仓库

https://docs.docker.com/registry/ 

```bash
#本地仓库地址localhost，端口5000，与dockerhub相同
$ docker run -d -p 5000:5000 --restart always --name registry registry:2
```

#### 推送镜像

```bash
#本地仓库地址地址打上标签
$ docker tag nginx:alpine localhost:5000/nginx:alpine

#推送镜像到本地仓库
$ docker push localhost:5000/nginx:alpine

$ docker images	
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE      
nginx                    alpine              333c0837328f        1 weeks ago         
localhost:5000/nginx     alpine              333c0837328f        2 weeks ago         
registry                 2                   708bc6af7e5e        1 months ago     
```

#### 登录

```bash
$ docker login HOST -u USER -p PASS
$ docker logout
```

### 容器相关

#### 容器查看

```bash
## 查看运行状态的容器列表
$ docker ps
## 查看全部状态的容器列表
$ docker ps -a
```

#### 启动

```bash
## 后台启动
$ docker run --name nginx -d nginx:alpine
## 映射端口,把容器的端口映射到宿主机中,-p <host_port>:<container_port>
$ docker run --name nginx -d -p 8080:80 nginx:alpine
## 资源限制,最大可用内存500M
$ docker run --memory=500m nginx:alpine

#进入容器或者执行容器内的命令
$ docker exec -ti <container_id_or_name> /bin/sh
$ docker exec <container_id_or_name> hostname
```

#### 停止

```bash
 ## 停止运行中的容器
 $ docker stop nginx
 ## 启动退出容器
 $ docker start nginx
 ## 删除非运行中状态的容器
 $ docker rm nginx
 ## 删除运行中的容器
 $ docker rm -f nginx
```

#### 日志查看

```bash
## 查看全部日志
$ docker logs nginx
 
## 实时查看最新日志
$ docker logs -f nginx
 
## 从最新的100条开始查看
$ docker logs --tail=100 -f nginx
```

#### 持久化

```bash
## 挂载主机目录
$ docker run --name nginx -d  -v /opt:/opt  nginx:alpine
$ docker run --name mysql -e MYSQL_ROOT_PASSWORD=123456  -d -v /opt/mysql/:/var/lib/mysql mysql:5.7
```

#### 数据拷贝

```bash
## 主机拷贝到容器
$ echo '123'>/tmp/test.txt
$ docker cp /tmp/test.txt nginx:/tmp
$ docker exec -ti nginx cat /tmp/test.txt
123
 
## 容器拷贝到主机
$ docker cp nginx:/tmp/test.txt ./
```

#### 明细

```bash
## 查看容器详细信息，包括容器IP地址等
$ docker inspect nginx

## 查看镜像的明细信息
$ docker inspect nginx:alpine
```

