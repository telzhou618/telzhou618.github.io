# k8s学习笔记 - k8s worker 节点扩容


在现实中，往往会遇到集群节点资源紧张，这时候需要增加更多的节点来缓解，并且k8s会自动迁移一些容器到新的节点上，以保证各个节点压力均很，接下来利用 kubesphere 社区提供的集群安装工具 kubekey 来给集群增加新的 worker 节点。

现有的集群有三个节点，分别为master、node1、node2， 现在给集群扩容一个节点 node3，操作步骤如下所示。

&lt;!--more--&gt;
### 创建新机器

集群用的 VM 虚拟机，系统centos7，先找一台干净的虚拟机克隆一台新机器，注意需要关闭VM后才能点击克隆。

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507155736.png)

### 配置网络

新克隆的虚拟机可能存在网络IP冲突，需要先配置一个新的IP地址，才能用xshell工具连接，按照如下步骤配置网络。

1. 在vm窗口登录root账号

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507160215.png)

2. 使用 nmtui 引导配置网络，在终端输入nmtui，具体步骤如下：

```shell
nmtui
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507161258.png)

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507161331.png)

3. 修改一个新的IP地址  

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507160506.png)

4. 保存后重启网络，如果没有任何输出说明ok。

```shell
systemctl restart newwork
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507161434.png)

5. 然后就可以用 xshell 连接了，操作起来更加方便。

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507161522.png)



### 安装依赖

1.  设置 hostname

```shell
hostnamectl set-hostnmae node3
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507161634.png)

2. 设置时区

```shell
timedatectl set-timezone Asia/Shanghai
timedatectl 
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507161845.png)

3. 配置时间同步器

```shell
yum install -y chrony
systemctl restart chronyd &amp;&amp; systemctl enable chronyd
```

4. 配置集群所有节点 hosts , 把 node3 加入到 hosts配置文件中。

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507162111.png)

5. 其他三个要修改的地方，和安装集群时操作一致，这里放在一起说明。

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
```

6. 安装依赖, 这是集群所必须依赖的软件。

```shell
# 安装 Kubernetes 系统依赖包
yum install curl socat conntrack ebtables ipset ipvsadm
# 安装其他必备包，openEuler 也是奇葩了，默认居然都不安装tar，不装的话后面会报错
yum install tar


```



### 执行扩容

1.  在 master 节点找到安装集群时用的配置文件和kk工具。

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507162608.png)

2. 执行 vim 修改配置文件，加入 node3 新节点的配置信息。

```shell
vim config-sample.yaml
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507162709.png)

3. 执行以下命令扩容，在检查通过后输入 yes

```shell
export KKZONE=cn

./kk add nodes -f config-sample.yaml
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507162746.png)

4. 一段时间后看到如下结果表示扩容成功。

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507162813.png)

### 验证状态

1. 查看集群的状态，发现node3已就绪。

```shell
kubectl get node
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507162902.png)

2. 登录到 kubesphere 控制台查看，部分节点已经自动迁移到 node3 上。

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507162914.png)

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240507163106.png)

### 参考文档

- [基于 KubeKey 扩容 Kubernetes v1.24 Worker 节点实战](https://kubesphere.io/zh/blogs/expanding-kubernetes-v1.24-worker-nodes-with-kubekey/)

---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/a9d9636/  

