# Docker-Compose 容器编排


## Docker-Compose

可以批量管理多个容器。

### 编写 docker-compose 文件

- 以Redis为例，执行 vi docker-compose-redis.yml。

```dockerfile
version: &#39;3&#39;
services:
  # redis
  redis:
    image: redis
    volumes:
      - /var/volumes/redis_data:/data
    restart: always
    ports:
      - &#34;6379:6379&#34;
```

### 启动容器

```sh
docker-compose -f docker-compose-redis.yml up -d
docker-compose -f docker-compose-redis.yml up -d --build // 每次重新打包新的镜像
```

- -f 指定compose 文件，默认查找docker-compose.yml.
- -d 后台启动。

### 配置文件常用参数

- image 指定镜像名称或ID

```yaml
image：java
```

- build 指定Dockerfile文件的路径

```yaml
 build: ./dir
```

- command 覆盖容器启动后默认执行的命令。
- links 连接其他容器

```yaml
web:
	links:
		- db
		- redis
```

- external_links 连接外部容器，格式和links一样。

- ports 暴露端口信息，格式：宿主主机端口：容器端口,只指定容器端口时宿主主机端口随机

```yaml
ports:
  - &#34;8081&#34;
	- &#34;8080:8080&#34;
```

- expose 只保留容器端口

```yaml
expose:
	- &#34;8000&#34;
	- &#34;9000&#34;
```

- volumes 卷挂在路径，格式：宿主主机路径:容器路径

```yaml
volumes:
	- /opt/data:/var/lib/mysql
```

- environment 设置环境变量

```yaml
environment:
	RACK_ENV dev
	SHOW: false
```

- net 设置网络

```yaml
net: &#34;bridge&#34; 
net: &#34;host&#34;
net: &#34;none&#34;
net: &#34;container:[service name or container name/id]&#34;
```

- dns 设置dns,可以一个，也可以是多个

```yaml
dns: 8.8.8.8
dns:
	- 8.8.8.8
	- 9.9.9.9
```

### docker-compose 常用操作命令

- 查看容器

```bash
docker-compose -f docker-compose.yml ps
```

- 关闭/启动/重启某个容器

```bash
docker-compose -f docker-compose.yml stop/start/restart &lt;服务名称&gt; // 不加服务名则会操作所有容器
```

- 查看容器日志

```bash
docker-compose -f docker-compose.yml logs -f 						 	// 查看所有容器日志
docker-compose -f docker-compose.yml logs -f	&lt;服务名&gt; 		// 查看指定容器日志
docker-compose -f docker-compose.yml logs -f &gt;&gt; app.log &amp; // 把日志输出到文件
```

- 重新构建镜像并启动

```bash
docker-compose -f docker-compose.yml up --build -d
```

- 重新构建 cokder-compose.yml 有变化的容器并启动

```bash
docker-compose -f docker-compose.yml up --fore-recreate -d
```

- 停掉容器并删除

```bash
docker-compose -f docker-compose.yml down
```

## prometheus &#43; grafana 搭建监控

### 安装 redis-exporter
```sh
docker run -d \
	--name redis_exporter \
	-p 9121:9121 \
	-v /etc/localtime:/etc/localtime:ro \
	oliver006/redis_exporter \
	--redis.addr redis://192.168.202.101:6379 \ # redis 地址
```

### 安装 prometheus

```sh
cd /opt
mkdir prometheus
cd /opt/prometheus
vi prometheus.yml
```

prometheus.yml

```yaml
global:
  scrape_interval: 5s
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: [&#39;localhost:9090&#39;]
        labels:
          instance: prometheus
 
  - job_name: redis
    static_configs:
      - targets: [&#39;192.168.202.101:9121&#39;] # redis_exporter 地址
        labels:
          instance: redis
```

docker 启动 prometheus

```sh
docker run  -d \
  -p 9090:9090 \
  --name=prometheus \
  -v /etc/localtime:/etc/localtime:ro \
  -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml  \
  prom/prometheus
```

### 安装grafana

```sh
cd /opt
mkdir grafana-storage
chmod 777 grafana-storage
```

```sh
# grafana
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  -v /opt/grafana-storage:/var/lib/grafana \
  -v /etc/localtime:/etc/localtime:ro \
  grafana/grafana
```





```sh
docker run -d --name node_exporter \
	-p 9100:9100 \
	--restart=always \
	--net=&#34;host&#34; \
	--pid=&#34;host&#34; \
	-v &#34;/proc:/host/proc:ro&#34; \
	-v &#34;/sys:/host/sys:ro&#34; \
	-v &#34;/:/rootfs:ro&#34; \
	prom/node-exporter \
	--path.procfs=/host/proc \
	--path.rootfs=/rootfs \
	--path.sysfs=/host/sys \
	--collector.filesystem.ignored-mount-points=&#39;^/(sys|proc|dev|host|etc)($$|/)&#39;
```



---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/docker-compose/  

