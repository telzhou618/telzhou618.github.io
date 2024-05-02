# Jenkins 的三种安装方式




## docker 安装JK

下载镜像

```shell
docker pull jenkins
```

启动

```shell
docker run -d -p  8088:8080 -p 50000:50000 -v /docker/jenkins_home:/var/jenkins_home jenkins
```

报错了，挂在的目录 /docker/jenkins_home 无权限
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240424004723.png)

解决方案：更改宿主机挂载目录的权限,使用 chmod 命令将 /docker/jenkins_home 目录的权限更改为允许容器内的用户写入

```shell
sudo chmod 777 /docker/jenkins_home
```

&gt;  注意：chmod 777 是开放最大权限，这样做可能会存在安全风险，最好根据实际需求设置更严格的权限。

再次执行上面的docker run 重启容器，然后 docker ps 查看运行状态,显示Up,正常。

```shell
docker ps 
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240424005631.png)


打开浏览器，输入宿主主机ip:8080 看到下面结果说明jk安装成功。
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240424005839.png)

登录jenkins, 根据界面提示密码在 /var/jenkins_home/secrets/initialAdminPassword,docker 启动时做了目录映射，对应宿主机的目录为 /docker/jenkins_home/

```shell
cat /docker/jenkins_home/secrets/initialAdminPassword 
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240424010359.png)
输入密码后登录。

## centos yum安装JK

### 安装 jdk

最新版的jk最低要求java11,先卸载低版本的 java, 查找一下已安装的java版本

```shell
yum list installed | grep java
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502002214.png)
查找 java1.7和1.8两个版本，都先卸载掉,执行以下命令

```shell
yum remove -y java*
```

然后安装 java11, 在安装之前先查找一下

```shell
yum list | grep java-11
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502001521.png)

执行 Install 安装

```shell
yum install -y java-11-openjdk-devel
```

&gt; 注意，执行 yum install -y java-11-openjdk 安装的的包不完整，会缺少jps、javac等命令，这里执行上面命令，会自动依赖安装 java-11-openjdk 包。

验证是否安装成功

```shell
java -version
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502002910.png)
ok, 下面安装jenkins。

### 安装 jenkins

设置官方最新jk镜像仓库,地址：https://pkg.jenkins.io/redhat-stable/

```shell
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
```

安装

```shell
 yum install -y jenkins
```

查看安装位置

```shell
rpm -ql jenkins
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502010517.png)

启动jk服务

```shell
systemctl start jenkins
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502003842.png)
日志显示启动失败,查看下状态

```shell
systemctl status jenkins
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502004026.png)
再看看启动日志

```shell
journalctl -u jenkins.service # 从头查看日志
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502004147.png)
发现端口被占用，这种情况会一直重启，不断尝试。

原来的是我的 docker 容器有个tomcat占用了8080端口，先把它kill或停止掉。

```shell
docker ps
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502004409.png)

```shell
docker stop my-tomcat # 停止tomcat容器
```

再次启动 jenkins

```shell
systemctl start jenkins # 启动
systemctl status jenkins # 查看状态
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502004749.png)

在浏览器输入ip:8080 查看，看到下面界面说明安装成功了。
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502004926.png)

根据提示密码存在 /var/lib/jenkins/secrets/initialAdminPassword 文件中，找到后输入密码。

安装插件页面，推荐选择“安装推荐的插件”，点击等等安装。
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502005419.png)

等等安装下面的插件。
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502005456.png)

下一步创建管理员用户。
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502005857.png)

点击完成，进入JK主界面。
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502010005.png)

### 开机启动

启动服务

```shell
systemctl start jenkins
```

查看服务状态

```shell
systemctl status jenkins
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502000844.png)
查看启动日志

```shell
journalctl -u jenkins.service # 从头查看日志
journalctl -f -u jenkins.service # -f 监听日志变化
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502000741.png)
查看服务的开机启动状态

```shell
systemctl list-unit-files | grep jenkins
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502000636.png)
设为开机启动

```shell
systemctl enable jenkins 
```

其他 systemctl 命令

```shell
systemctl stop jenkins  # 停止
systemctl restart jenkins # 重启
systemctl enable jenkins # 设为开机启动
systemctl disable jenkins # 禁用开机启动
```

## war 包安装JK

下载 war 包，下载地址：https://www.jenkins.io/zh/doc/book/installing/，点击WAR包下载。

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502103240.png)

前提安装java11,新版jk的最低要求。

最简单的方式启动

```
java -jar jenkins.war
```

&gt; 默认端口8080

指定端口启动

```shell
java -jar jenkins.war --httpPort=9090
```

**生产环境最佳实战**

1. 创建工作目录

```shell
mkdir -p /root/.jinkins_home
```

2. 启动jk服务

```shell
java -DJENKINS_HOME=/root/.jenkins_home/ -jar jenkins.war --httpPort=9090 --logfile=/root/.jenkins_home/jenkins.log
```

&gt; - -DJENKINS_HOME 指定工作目录
&gt; 
&gt; - --httpPort 端口
&gt; 
&gt; - --logfile 日志文件位置

3. 后台启动

```shell
nohup java -DJENKINS_HOME=/root/.jenkins_home/ -jar jenkins.war --httpPort=9090 --logfile=/root/.jenkins_home/jenkins.log &amp;
```

&gt; 会在当前目录下生成 nohup.out 文件作为输出日志，更多详细日志查看--logfile 的配置。

4. 浏览器地址输入 ip:9090 验证。


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/704c8ae/  

