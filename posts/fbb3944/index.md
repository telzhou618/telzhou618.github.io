# k8s学习笔记 - Kubekey安装k8s和kubesphere


![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507164822.png)



## Kubekey 一键安装k8s集群
&lt;!--more--&gt;

### 准备工作

三台虚拟机, 系统为 centos7, 如下:

| 机器IP         | hostname   | 角色     |
| ------------ | ---------- | ------ |
| 192.168.1.20 | k8s-master | master |
| 192.168.1.21 | k8s-node1  | worker |
| 192.168.1.22 | k8s-node2  | worker |


执行如下操作前置操作

```shell
1、关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

2、关闭 selinux
sed -i &#39;s/enforcing/disabled/&#39; /etc/selinux/config # 永久关闭
setenforce 0 # 临时关闭

3、关闭 swap
swapoff -a # 临时关闭
vim /etc/fstab # 永久关闭
#注释掉swap这行
# /dev/mapper/centos-swap swap                    swap    defaults        0 0

systemctl reboot  #重启生效
free -m  #查看下swap交换区是否都为0，如果都为0则swap关闭成功

4、给三台机器分别设置主机名
hostnamectl set-hostname &lt;hostname&gt;
第一台：k8s-master
第二台：k8s-node1
第三台：k8s-node2

5、设置时间同步
yum install ntpdate -y
ntpdate time.windows.com
```

### kubekey安装k8s集群

1. 安装 socat，必须

```shell
yum install -y socat
```

2. 安装conntrack，必须

```shell
yum install -y conntrack
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506232906.png)



3. 下载 kubekey 工具

```shell
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.13 sh -
```

4. 授权kk工具可执行权限

```shell
chmod &#43;x kk
```

5. 创建集群配置文件

```shell
./kk create config --with-kubernetes v1.23.10 --with-kubesphere v3.4.1
```

&gt; 在当前目录生成  config-sample.yaml
&gt; 
&gt; 可通过  [-f ~/myfolder/abc.yaml] 制定配置文件位置

6. 编辑配置文件，修改 hosts 节点信息，修改 roleGroups 各个组件的节点

```shell
 vim config-sample.yaml 
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506195049.png)

7. 创建集群

```shell
./kk create cluster -f config-sample.yaml
```

&gt; 验证通过会提是是否开始安装，输入 yes 暗转，耐心等待
&gt; 
&gt; 会自动安装依赖的 docker 机器 k8s相关的组件，同时会安装 kubesphere 管理工具。

8. 一段时间后看到类似下面界面表示安装成功，速度取决于网速

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506195352.png)

9. 查看集群，浏览区输入上面的地址和账号密码登录

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506232253.png)


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/fbb3944/  

