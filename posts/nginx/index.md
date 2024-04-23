# CentOS 安装和使用最新版 Nginx


## 安装最新版 Nginx

配置 EPEL 源,否则安装的是旧的或找不到Nginx源
```shell
sudo yum install -y epel-release
```
安装Nginx
```shell
sudo yum install -y nginx
```

启动 Nginx
```shell
systemctl start nginx
```
访问: http://localhost, Nginx 默认端口是80，可以省略
![](https://raw.githubusercontent.com/telzhou618/images/main/img03/img.png)


## Nginx 常用命令
```shell

# 检查 nginx 配置是否正确
nginx -t

# 停止服务
nginx -s stop

# 退出服务
nginx -s quit

# 重启服务
nginx -s reload
```
## 用 systemctl 操作

1.启动 Nginx
```shell
systemctl start nginx
```
2.停止 Nginx
```shell
systemctl stop nginx
```
3.重启 Nginx
```shell
systemctl restart nginx
```
4.查看 Nginx 状态
```shell
systemctl status nginx
```

5.启用开机启动 Nginx
```shell
systemctl enable nginx
```

6.禁用开机启动 Nginx
```shell
systemctl disable nginx
```

## 配置域名转发

cd 到 /etc/nginx/conf.d 目录,创建配置文件，创建后缀为conf的文件,这里以gitlab服务为例，假如 gitlab 的IP端口为 http://192.168.1.4:10008
```shell
cd /etc/nginx/conf.d
vi www.gitlab.net.conf
```
键入以下配置
```shell
server {
    listen 80;
    server_name  www.gitlab.net;

    location / {
        proxy_pass http://192.168.1.4:10008;
    }
}
```
保存,重启 Nginx
```shell
# 测试配置
nginx -t  
# 无报错重启
nginx -s reload 
```
修改本机 hosts 测试
```shell
192.168.1.4 www.gitlab.net
```
访问 www.gitlab.net 测试,ok!

![](https://raw.githubusercontent.com/telzhou618/images/main/img03/20240423004633.png)







---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/nginx/  

