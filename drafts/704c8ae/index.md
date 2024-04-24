# Jenkins


&lt;!--more--&gt;

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
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240424004723.png)

解决方案：更改宿主机挂载目录的权限,使用 chmod 命令将 /docker/jenkins_home 目录的权限更改为允许容器内的用户写入
```shell
sudo chmod 777 /docker/jenkins_home
```
&gt;  注意：chmod 777 是开放最大权限，这样做可能会存在安全风险，最好根据实际需求设置更严格的权限。

再次执行上面的docker run 重启容器，然后 docker ps 查看运行状态,显示Up,正常。
```shell
docker ps 
```
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240424005631.png)

打开浏览器，输入宿主主机ip:8080 看到下面结果说明jk安装成功。
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240424005839.png)

登录jenkins, 根据界面提示密码在 /var/jenkins_home/secrets/initialAdminPassword,docker 启动时做了目录映射，对应宿主机的目录为 /docker/jenkins_home/
```shell
cat /docker/jenkins_home/secrets/initialAdminPassword 
```
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240424010359.png)
输入密码后登录




---

> 作者:   
> URL: https://telzhou618.github.io/drafts/704c8ae/  

