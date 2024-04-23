# Otter数据同步 - 环境搭建


{{&lt; figure src=&#34;https://raw.githubusercontent.com/telzhou618/images/main/img02/1686739961279-58ffcfde-b8b0-4a9b-af4a-7630d29dd839.png&#34; title=&#34;Otter架构&#34; &gt;}}


## otter 官网介绍
名称：otter [&#39;ɒtə(r)]
译意： 水獭，数据搬运工
语言： 纯java开发
定位： 基于数据库增量日志解析，准实时同步到本机房或异地机房的mysql/oracle数据库. 一个分布式数据库同步系统。
更多介绍查看Github：[https://github.com/alibaba/otter](https://github.com/alibaba/otter)

## 架构及工作原理
1. Canal, 负责监听Bin-log，类似MYSQL的从库，本质上实现了MYSQL的协议，伪装成从库读取bin-log日志。
2. Manager 管理Otter的配置的后台, 比如管理数据库表之间的映射关系，会把配置推送到 Node。
3. Node 实现同步功能的核心服务，同时把同步结果实时反馈给Manager。
4. Zookeepr, 分布式调度，监控节点状态，这个不用解释，分布式系统都会用到。

otter 需要以上四个服务才能工作，搭建比较麻烦，稍后详细说明。
## 本次目标
假设要把tb_user表的数据同步到tb_user2, 用otter该如何实现？tb_user 表结构如下。
```shell
create table tb_user
(
    id            int(11) unsigned auto_increment comment &#39;ID&#39;
        primary key,
    username      varchar(100) default &#39;&#39;                not null comment &#39;用户名&#39;,
    age           int          default 0                 not null comment &#39;年龄&#39;,
    email         varchar(64)  default &#39;&#39;                not null comment &#39;邮箱&#39;,
    created_time  datetime     default CURRENT_TIMESTAMP not null comment &#39;创建时间&#39;,
    modified_time datetime     default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment &#39;更新时间&#39;,
    is_deleted    tinyint(2)   default 0                 not null comment &#39;是否删除&#39;
)
    comment &#39;用户信息表&#39; charset = utf8mb4;
```
tb_user2 表结构和tb_user相同。
```shell
create table tb_user2 like tb_user;
```
接下来搭建同步所需的各个服务。
## 安装 zookeeper
打开ZK官网，下载最新的包，地址：[https://zookeeper.apache.org/releases.html#download](https://zookeeper.apache.org/releases.html#download)
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686741059427-c3f9fe02-99ce-4760-b5d1-5e1ed22bee9b.png)
执行下面命令解压文件
```shell
tar -zxvf apache-zookeeper-3.8.1-bin.tar.gz
```
解压后得到如下目录
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1687943180670-d6a3da1b-847a-4ffb-803f-dbb6a99ab373.png)
进入到conf配置目录修改配置文件，拷贝 zoo_sample.cfg 文件重名名为 zoo.cfg
```shell
cd conf
cp zoo_sample.cfg zoo.cfg zoo.cfg
```
vi 修改配置
```shell
vi zoo.cfg
```
主要看下面两个配置，zk 的默认端口是 2181，如果没有被占用，可以不改。
```shell
# example sakes.
dataDir=/tmp/zookeeper # zk数据存储目录
# the port at which the clients will connect
clientPort=2181 # 端口默认2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
```
启动zk
```shell
sh bin/zkServer.sh
```
连接zk
```shell
bin/zkCli.sh
```
看到如下所示说明zk安装成功。
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686741630437-6862575e-3283-40eb-a71f-caf15a89db49.png)
## 安装 Canal
下载最新文档的 Canal,下载地址：[https://github.com/alibaba/canal/releases](https://github.com/alibaba/canal/releases)
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1687943666596-1da70142-215d-4c04-8ecc-d1ae7cdfd7e3.png)
解压文件
```shell
tar -zxvf canal.deployer-1.1.6.tar.gz
```
得到如下目录
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1687943583010-1f13be9e-9e31-45dd-b46d-f546472ae706.png)
修改配置,参考官网文档：[https://github.com/alibaba/canal/wiki/QuickStart](https://github.com/alibaba/canal/wiki/QuickStart)
对于自建 MySQL , 需要先开启 Binlog 写入功能，配置 binlog-format 为 ROW 模式，my.cnf 中配置如下
```
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式
server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
```

- 注意：针对阿里云 RDS for MySQL , 默认打开了 binlog , 并且账号默认具有 binlog dump 权限 , 不需要任何权限或者 binlog 设置,可以直接跳过这一步
- 授权 canal 链接 MySQL 账号具有作为 MySQL slave 的权限, 如果已有账户可直接 grantCREATE USER canal IDENTIFIED BY &#39;canal&#39;;   GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO &#39;canal&#39;@&#39;%&#39;; -- GRANT ALL PRIVILEGES ON *.* TO &#39;canal&#39;@&#39;%&#39; ; FLUSH PRIVILEGES;
```shell
vi conf/example/instance.properties
```
```shell
## mysql serverId
canal.instance.mysql.slaveId = 1234
#position info，需要改成自己的数据库信息
canal.instance.master.address = 127.0.0.1:3306 
canal.instance.master.journal.name = 
canal.instance.master.position = 
canal.instance.master.timestamp = 
#canal.instance.standby.address = 
#canal.instance.standby.journal.name =
#canal.instance.standby.position = 
#canal.instance.standby.timestamp = 
#username/password，需要改成自己的数据库信息
canal.instance.dbUsername = canal  
canal.instance.dbPassword = canal
canal.instance.defaultDatabaseName =
canal.instance.connectionCharset = UTF-8
#table regex
canal.instance.filter.regex = .\*\\\\..\*
```
启动
```shell
sh bin/startup.sh
```
查看日志
```shell
vi logs/canal/canal.log&lt;/pre&gt;
```
```shell
2013-02-05 22:45:27.967 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## start the canal server.
2013-02-05 22:45:28.113 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[10.1.29.120:11111]
2013-02-05 22:45:28.210 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## the canal server is running now ......
```
## 安装 Manager

1. 先下载安装包，下载地址：[https://github.com/alibaba/otter/releases](https://github.com/alibaba/otter/releases)

![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1687943785980-a9ebdf60-803f-41c3-a06f-3255db1b273a.png)解压
```shell
tar -zxvf manager.deployer-4.2.18.tar.gz
```
得到下面目录
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686801427795-b8dedf0f-eeb3-4cdb-9590-6aa4c93cbab5.png)

3. manager 需要数据库，下载官方SQL脚本，并导入进行数据库表创建和数据初始化
```shell
wget https://raw.github.com/alibaba/otter/master/manager/deployer/src/main/resources/sql/otter-manager-schema.sql 
```
```shell
source otter-manager-schema.sql
```

4. 修改配置，编辑conf目录下的otter.properties 配置文件，修改如下所示
```shell
## otter manager domain name #修改为正确访问ip，生成URL使用
otter.domainName = 127.0.0.1    
## otter manager http port
otter.port = 8080
## jetty web config xml
otter.jetty = jetty.xml

## otter manager database config ，修改为正确数据库信息
otter.database.driver.class.name = com.mysql.jdbc.Driver
otter.database.driver.url = jdbc:mysql://127.0.01:3306/ottermanager
otter.database.driver.username = root
otter.database.driver.password = hello

## otter communication port
otter.communication.manager.port = 1099

## otter communication pool size
otter.communication.pool.size = 10

## default zookeeper address，修改为正确的地址，手动选择一个地域就近的zookeeper集群列表
otter.zookeeper.cluster.default = 127.0.0.1:2181
## default zookeeper session timeout = 90s
otter.zookeeper.sessionTimeout = 90000

## otter arbitrate connect manager config
otter.manager.address = ${otter.domainName}:${otter.communication.manager.port}
```
4. 启动
```shell
sh startup.sh
```
5. 查看日志
```shell
2013-08-14 13:19:45.911 [] WARN  com.alibaba.otter.manager.deployer.JettyEmbedServer - ##Jetty Embed Server is startup!
2013-08-14 13:19:45.911 [] WARN  com.alibaba.otter.manager.deployer.OtterManagerLauncher - ## the manager server is running now ......
```

6. 验证

访问： http://127.0.0.1:8080/，出现otter的页面，即代表启动成功
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686810971785-c6851c60-e2bb-4ab6-8f2c-245d1b8e0887.png)
输入账号密码，默认amdin/admin
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686811010972-58dfc798-272a-4547-8364-f725dde7f1de.png)
注意，因为我的8080端口被占用，这里我改成8082，你可以根据你的情况修改端口，我截图中有两个Channel,这是我测试创建的，刚安装完是什么空的，什么都没有。
一切 OK ！

7. 把前面安装的ZK服务添加到manger,后面安装Node等需要用到ZK。

点击manager界面的机器管理&gt;zookeeper管理。
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686822045989-b3393b3e-232d-4744-a68b-7a71f55b1db9.png)然后添加,输入ZK集群的信息，集群名称随意，集群地址填ZK的IP和端口，中间用冒号分隔，形如“ip:端口”的新式，多个分号隔开。
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686811837736-ceaedac0-d008-492b-90a2-ea458a6a102b.png)
点击保存，OK。
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686822220656-9ad24b18-fd41-4f9b-acdc-3fde8c523163.png)
## 安装 Node
下载官方安装包，地址：[https://github.com/alibaba/otter/releases](https://github.com/alibaba/otter/releases)  和上面的manager同一个地址，下载如下文件。
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686811178653-a7e2bbb6-daca-4d8d-8da4-0a9133a26ea2.png)
解压缩，得到如下目录
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686811212433-512a265c-9e81-4486-905b-591bedbf6ca5.png)
这时需要回到manger添加一个Node, 得到Node的序号，因为启动Node服务时需要这个序号。
在manger界面选择机器管理&gt;Node管理
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686811411295-42f01dc2-e195-4bc1-9600-bf9e938df04d.png)
在Node管理页面点击添加，输入机器名称、IP和端口，以及选择一个ZK集群，这个几个选项是必填的，机器名称随意，机器IP输入要启动Node服务的IP，端口就是Node服务的端口，默认2088。
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686811643422-14fe2535-7d21-4023-9d06-de6c3e5e7767.png)
保存后得到下面结果，这里有个关键信息有两个，一个是状态，刚添加完为启动Node是状态是未启动，还有一个是序号，序号是点击保存是系统生成的，在后面配置文件中会用到，我这里是1
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1686811961934-07564080-4fb8-4f92-93bf-ddec0babb624.png)
修改Node的配置
首先在conf目录下创建一个nid的文件，值为前面生成的序号，如下所示 (将环境准备中添加机器后获取到的序号，保存到conf目录下的nid文件，比如我添加的机器对应序号为1)
```shell
echo 1 &gt; conf/nid
```
然后修改 otter.properties配置
```shell
# otter node root dir
otter.nodeHome = ${user.dir}/../node 
## otter node dir
otter.htdocs.dir = ${otter.nodeHome}/htdocs
otter.download.dir = ${otter.nodeHome}/download
otter.extend.dir= ${otter.nodeHome}/extend

## default zookeeper sesstion timeout = 90s
otter.zookeeper.sessionTimeout = 90000

## otter communication pool size
otter.communication.pool.size = 10

## otter arbitrate &amp;amp; node connect manager config ， 修改为正确的manager服务地址
otter.manager.address = 127.0.0.1:1099
```
启动
```shell
sh startup.sh
```
查看日志
```shell
vi logs/node/node.log
```
```shell
2013-08-14 15:42:16.886 [main] INFO  com.alibaba.otter.node.deployer.OtterLauncher - INFO ## the otter server is running now ......

```
验证
再次访问manger 页面， [http://127.0.0.1:8080/node_list.htm](http://127.0.0.1:8080/node_list.htm)，查看对应的节点状态，如果变为了已启动，代表已经正常启动。(ps，如果是未启动，会是一个红色高亮)

现在 otter 所需要的各个服务已准备好，接下来配置数据同步。




---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/otter1/  

