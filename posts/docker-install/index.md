# CentOS 安装Docker


## docker 安装

卸载旧的docker
```shell
sudo yum remove -y docker*
```
安装 yum-utils
```shell
yum install -y yum-utils
```
设置docker 源
```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
```
查看docker所有版本
```shell
yum list docker-ce --showduplicates | sort -r
```
安装docker
```shell
yum install -y docker-ce-3:19.03.9-3.el7.x86_64
```
设置为开机启动
```shell
systemctl start docker 
systemctl enable docker
```
验证是否成功
```shell
dokcer version
```
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240423101515.png)

修改阿里云 docker 镜像源加速，[https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240423101725.png)

```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json &lt;&lt;-&#39;EOF&#39;
{
  &#34;registry-mirrors&#34;: [&#34;https://0qnvd2nu.mirror.aliyuncs.com&#34;]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 部署 Tomcat
拉取镜像
```shell
docker pull tomcat:7.0.59
```
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240423103421.png)
查看镜像
```shell
docker images
```
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240423103456.png)
启动容器
```shell
docker run -d -p 8080:8080  --name my-tomcat tomcat:7.0.59
```
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240423103612.png)
访问测试, http://192.168.1.5:8080
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240423104056.png)

## 常用命令

当然！以下是从提供的 shell 脚本中提取出的各个 Docker 命令：

1. **拉取 Docker 镜像:**
```shell
docker pull java:8
```

2. **查看本地 Docker 镜像:**
```shell
docker images
```

3. **删除本地 Docker 镜像:**
```shell
docker rmi java
```

4. **删除所有本地 Docker 镜像:**
```shell
docker rmi $(docker images -q)
```

5. **运行 Docker 容器:**
```shell
docker run -d -p 8080:8080 --name my-tomcat tomcat:7.0.59
```

6. **查看正在运行的 Docker 容器:**
```shell
docker ps
```

7. **查看所有 Docker 容器（包括停止的）:**
```shell
docker ps -a
```

8. **停止 Docker 容器:**
```shell
docker stop 7c5721a009cc
```

9. **强制停止 Docker 容器:**
```shell
docker kill 7c5721a009cc
```

10. **启动 Docker 容器:**
```shell
docker start 7c5721a009cc
```

11. **查看 Docker 容器内部信息:**
```shell
docker inspect 7c5721a009cc
```

12. **查看 Docker 容器日志:**
```shell
docker container logs 7c5721a009cc
```

13. **监控 Docker 容器日志:**
```shell
docker container logs 7c5721a009cc -f
```

14. **查看 Docker 容器内进程:**
```shell
docker top 7c5721a009cc
```

15. **复制文件到/从 Docker 容器:**
```shell
docker cp 7c5721a009cc:/etc/nginx/nginx.conf /mydata/nginx  # 从容器复制到主机
docker cp /mydata/nginx/nginx.conf 7c5721a009cc:/etc/nginx/  # 从主机复制到容器
```

16. **进入 Docker 容器 Shell:**
```shell
docker exec -it 7c5721a009cc /bin/bash
```

17. **删除 Docker 容器:**
```shell
docker rm 7c5721a009cc  # 只能删除已停止的容器
docker rm -f 7c5721a009cc  # 可以强制删除正在运行的容器
```

18. **强制删除所有 Docker 容器:**
```shell
docker rm -f $(docker ps -a -q)
```

19. **从 Dockerfile 构建 Docker 镜像:**
```shell
docker build -t demo:1.0.0 .
```
19. **查看容器的CPU、内存**
```shell
docker stats
```

这些命令用于在命令行界面有效地管理 Docker 容器、镜像及相关操作。每个命令都在 Docker 生态系统中具有特定的用途。

## Dockerfile 编写

![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240423110348.png)
构建 spring-boot 项目镜像
```shell
From java:8
ADD target/demo.jar /app.jar
EXPOSE 8080
ENTRYPOINT java ${JAVA_OPTS} -jar /app.jar

```
&lt;!--more--&gt;







---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/docker-install/  

