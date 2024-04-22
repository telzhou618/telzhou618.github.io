# Linux 常用命令

## less

从文件开头查看少量内容命令，命令格式：less [参数] 文件 。

参数：
```markdown
- /字符串：向下搜索
- ?字符串：向上搜索
- n：重复前一个搜索(向下搜索)
- N：反向重复前一个搜索(向上搜索)
- 翻一页
    - f：前进一页
    - b：回退一页
- 翻半页：
    - d：前进半页
    - u：后退半页
- 翻一行：
    - e/j：前进一行
    - y/k：后退一行
```

实例1：查看日志, -N 显示行号
```shell
less -N /var/a.log
```
实例2：跳转到最后一行，并滚动打印。
```shell
shift &#43; f
```
操作1：在滚动打印的基础上停止滚动
```shell
control &#43; c
```

操作2：向上搜索关键词
```shell
?kewwaord &#43; enter
按n继续向上搜索相同字符串
按N继续向下搜索相同字符串
```

操作3：向下搜索关键词
```shell
/kewwaord &#43; enter
按N继续向下搜索相同字符串
按n继续向上搜索相同字符串
```
## chown

设置文件所有者对文件的权限。

实例1：给admin用户授于/var/log.a.log文件权限
```shell
chown admin /var/log.a.log
```

实例2：递归给某个用户授于整个目录权限
```shell
chown -R admin /var/log/
```

## chmod

设置用户对文件的读、写、执行权限。
![image.png](https://raw.githubusercontent.com/telzhou618/images/main/img02/1644229447505-8d572e68-0426-466e-a750-330c05e36803.png)

命令格式：
```shell
chmod [ugo&#43;rwx] file
```

参数：

- u - 文件拥有者、g - 拥有者同组的其他人、o - 其他用户、a - 所有用户。
- r - 读、w - 写、x - 执行
- -R 递归收取目录和子目录下所有文件权限

实例1：将文件 file1.txt 设为所有人皆可读取

```shell
chmod ugo&#43;r file1.txt
# 等价
chmod a&#43;r file1.txt
```
实例2：所有文件所有人可读。
```shell
chmod -R a&#43;r *
```

实例3：所有人读写执行
```shell
chmod 777 file
# 等价
chmod a=rwx file
```

## lsof
查看端口占用情况。

实例1：查看 占用9876 端口的进程。

```shell
lsof -i tcp:9876

# 执行结果
COMMAND   PID       USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    71184 zhougaojun   69u  IPv6 0xd4d49114bb1da0f5      0t0  TCP *:sd (LISTEN)
```

实例2：查找出正在监听的端口

```shell
lsof -i:80 -sTCP:LISTEN
lsof -i:80 | grep LISTEN
```
实例3：另外可以用java自带的jps命令查看java应用进程

```shell
jps

#执行结果
21873 Main
71184 NamesrvStartup
```

## rz

上传本地文件到远程linux服务器。

```shell
rz # centos 输入rz命令唤回车起文件选择弹框
```
## sz
下载服务文件到本地。

```shell
sz a.txt	# 下载文件a.txt到本地机器。
```

## scp

远程和本地之间传输文件, 加-r 拷贝整个文件夹。

实例1：考配置远程服务器tomct.log文件到本地指定位置

```shell
scp root@192.168.1.100:/var/log/tomcat/tomcat.log  /home/myfile/ 

scp -r root@192.168.1.100:/data/ /home/myfile/ # 拷贝文件夹到本地服务
```

实例2：拷贝本地文件到远程服务器

```shell
scp /home/myfile/test.txt root@192.168.1.100:/data/ # 单文件拷贝

scp /home/myfile/test1.txt test2.cpp test3.bin test.* root@192.168.1.100:/data/ # 多文件拷贝

scp -r /home/myfile/ root@192.168.1.100:/data/ # 拷贝文件夹
```

## find

查找文件。

实例1：在home目录下查找名为test的文件。
```shell
find /home -name test
```
实例2：在home目录下查找以test开头的文件。
```shell
find /home -name &#34;test*&#34;
```

实例3：在home目录下查找大小大于100k的文件
```shell
find /home -size &#43;100k
```

## netstat

显示各种网络信息，如查看TCP连接信息。

参数：

- a (all)显示所有选项，默认不显示LISTEN相关
- t (tcp)仅显示tcp相关选项
- u (udp)仅显示udp相关选项
- n 拒绝显示别名，能显示数字的全部转化成数字。
- l 仅列出有在 Listen (监听) 的服務状态
- p 显示建立相关链接的程序名
- r 显示路由信息，路由表
- e 显示扩展信息，例如uid等
- s 按各个协议进行统计
- c 每隔一个固定时间，执行该netstat命令。

实例：查看所有TCP连接的端口和进程信息。
```shell
netstat -antp
```

## ps

显示进程的状态，实例。
```shell
ps -ef	# 显示进程及进程启动详细情况。
ps -aux	# 线程带CPU占用、内存信息等详细情况。
```
其他参数：

- a：显示一个终端的所有进程，除会话引线外；
- u：显示进程的归属用户及内存的使用情况；
- x：显示没有控制终端的进程；
- l：长格式显示更加详细的信息；
- e：显示所有进程；
## ln
生成链接文件，实例。
```shell
ln -s  source_file target_file	# 生成软连接，也称为符号链接，相当于快捷方式。
ln source_file target_file		# 生成硬链接，也称为实体链接，相当于拷贝。
```

## top
监控CPU、内存、负载、线程等运营情况

实例1: 基本用法
```shell
top 	#	按CPU占用排序，然后按sheft &#43; p
top 	#	按内存占用排序。然后按 sheft &#43; m
top -d 2 	# 每个2秒执行一次top。
```
示例2：查看指定进程下的所有线程。
```shell
top -H -p [pid] # 方式1，直接一步完成。
top -p [pid]    # 方式2，执行完需要按大写H。
```

## htop
以一种更友好的方式监控机器的运行状况，安装命令如下。
```shell
yum install htop -y  # centos

brew install htop  # mac
```
实例：直接执行htop可以监控CPU、内存、线程情况，并且以高亮的新式展示。
![](https://raw.githubusercontent.com/telzhou618/images/main/img02/image%20(4).png)


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/linux/  

