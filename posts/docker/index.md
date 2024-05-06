# Docker 入门

##  Docker 架构

{{&lt; figure src=&#34;https://raw.gitmirror.com/telzhou618/images/main/img/image-20210810154908558.png&#34; title=&#34;Docker架构&#34; &gt;}}


Docker 主要组件包括 Client、Daemon、Images、Container、Registry。

- Client ： docker客户端
- Daemon：docker服务端守护进程。
- Images：docker镜像，有点像jar包。
- Container：运行docker镜像的容器，像Tomcat。
- Registry：docker镜像仓库，像maven仓库。

## Docker 常用命令

### 镜像相关的命令

#### 搜索镜像

```sh
docker search java  // 搜索镜像
```

#### 下载镜像

```sh
docker pull java:8  # 下载镜像
```

#### 查看镜像

```sh
docker images  # 查看镜像
```

#### 删除镜像

```sh
docker rmi java  # 删除镜像，加-f强制删除
docker rmi $(docker images ‐q) # 删除所有镜像
```

### 容器相关的命令

#### 运行Docker镜像

```sh
docker run -d -p 8080:80 nginx:least
```

- -d 后台运行
- -p 指定端口
  - 如：8080:8080，宿主主机端口：容器端口
- -m 指定容器的内存大小
  - 如 ：-m 500M
- -e 指定环境变量
  - 如：-e  JAVA_OPTS=&#39;‐Xms1028M ‐Xmx1028M ‐Xmn512M ‐Xss512K ‐XX:MetaspaceSize=256M ‐XX:MaxMetaspaceSize= 256M&#39; 
- -net 设置网络
  - --net=bridge 桥接，默认
  - --net=host 容器使用宿主网络，不安全。
  - --net=container=容器ID，使用和其他容器一样的网络
- -v 指定挂在目录
  - 如: /var/nginx/logs : /nginx/logs , 宿主主机：容器主机。

#### 列出容器列表

```sh
docker ps  # 加 -a 可列出所有的容器，包含停止的容器
```

#### 停止容器

```sh
docker stop [容器ID]  // 停止容器
docker kill [容器ID]  // 强制停止容器
```

#### 启动容器

```sh
docker start [容器ID]
```

#### 查看日志

```sh
docker container logs [容器ID]
```

#### 查看容器信息

```sh
docker inspect [容器ID]
```

#### 查看容器里的进程

```
docker top [容器ID]
```

#### 文件传输

```
docker cp [容器ID]:/容器文件路径 宿主主机文件路径	// 从容器cp到主机
docker cp 宿主主机文件路径 [容器ID]:/容器文件路径	// 从主机cp到容器
```

#### 进入到容器的 shell

```sh
docker exec -it [容器ID] /bin/bash
```

#### 删除容器

```sh
docker rm [容器ID] // 删除已停止的容器
docker rm -f [容器ID] // 强制删除正在运行的容器
docker rm ‐f $(docker ps ‐a ‐q) // 强制删除所有容器
```

## Docker 构建镜像

### Dockerfile 命令

- FROM 指定基础镜像，如 java:8
- RUN 构建镜像执行的命令。
- ADD 文件复制
- COPY 文件复制，不支持URL和压缩。
- CMD 容器启动命令。
- EXPOSE 容器暴露端口。
- WORKID 容器工作路径。
- ENV 环境变量。
- ENTRPINT 和CMD类似。
- VOLUME 指定存储目录，如：VOLUME[&#34;/data&#34;]。

### 编写Dockerfile

执行 vi Dockerfile编写镜像文件

```dockerfile
# 基础镜像
FROM java:8
# 复制文件
ADD java-demo.jar /app.jar
# 暴露端口
EXPOSE 8080
# 运行程序
CMD java -jar /app.jar
```

### 构建镜像

在Dockerfile 文件所在的目录执行以下命令构建镜像。

```
docker build -t java-demo:0.0.1 .
```

- -t 指定镜像的名称和版本，不指定版本默认为latest
- &#39;.&#39; 点代表Dockerfile文件的位置

### 上传到Docker镜像仓库

- 在docker hub 注册账号
- 在控制台使用 docker login 命令登录
- 给镜像打一个tag分组名称, 如： docker tag java-demo.jar:0.0.1  zhangsan/java-demo.jar:0.0.1
- push到远程仓库, 如：docker push zhangsan/java-demo.jar



---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/docker/  

