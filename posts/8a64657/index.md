# 【k8s学习笔记】 k8s安装和基本用法


&lt;!--more--&gt;

## 安装 k8s

准备三台机器, 系统为 centos7, 如下

| 机器IP         | hostname   | 角色     |
| ------------ | ---------- | ------ |
| 192.168.1.20 | k8s-master | master |
| 192.168.1.21 | k8s-node1  | worker |
| 192.168.1.22 | k8s-node2  | worker |

### 准备工作

安装前期准备，以下8个步骤在三台机器都要执行。

1. 关闭防火墙

```shell
systemctl stop firewalld
systemctl disable firewalld
```

2. 关闭 selinux

```shell
sed -i &#39;s/enforcing/disabled/&#39; /etc/selinux/config # 永久关闭
setenforce 0 # 临时关闭
```

3. 关闭 swap

```shell
vim /etc/fstab # 永久关闭
#注释掉swap这行，需要重启生效
# /dev/mapper/centos-swap swap                    swap    defaults 
```

4. 设置机器hostname

```
hostnamectl set-hostname &lt;hostname&gt;

192.168.1.20：k8s-master
192.168.1.21：k8s-node1
192.168.1.22：k8s-node2
```

5. 设置时间同步

```shell
# 设置时区为亚洲上海
timedatectl set-timezone Asia/Shanghai
timedatectl status # 查看状态
# 同步时间
yum install ntpdate -y
ntpdate time.windows.com
```

&gt; 这一步成败的关键，三台机器的时间要同步

6. 重启机器,让以上所有设置生效

```shell
reboot
```

7. 安装 docker, 这是k8s所需要的

```shell
# 安装utiles 拥有 yum-config-manager 工具
yum install -y yum-utils
# 设置 docker yum 源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
# 查看 docker-ce 版本
yum list docker-ce --showduplicates | sort -r
# 安装
yum install -y docker-ce-3:19.03.9-3.el7.x86_64 # 这是指定版本安装
开启启动
systemctl start docker &amp;&amp; systemctl enable docker
```

8. 安装 kubelet, kubeadm, kubectl

```shell
# 添加 k8s yum 源
cat &gt; /etc/yum.repos.d/kubernetes.repo &lt;&lt; EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF


# 查看所有版本
yum list kubelet --showduplicates | sort -r

# 安装kubelet、kubeadm、kubectl 指定版本
yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0

# 开机启动
systemctl enable kubelet &amp;&amp; systemctl start kubelet
```

### 初始化集群

1. 初始化集群，在 master 节点执行

```shell
kubeadm init --apiserver-advertise-address=192.168.1.20 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.18.0 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16
```

2. 根据初始化结果执行以下操作，在 master 节点执行

```shell
#配置使用 kubectl 命令工具(类似docker这个命令)，执行上图第二个红框里的命令
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#查看kubectl是否能正常使用
kubectl get nodes

#安装 Pod 网络插件
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
# 如果上面这个calico网络插件安装不成功可以试下下面这个
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kubeflannel.yml
```

3. 将worker节点加入到集群，分别在node1和node2上执行以下操作

```shell
# 将node节点加入进master节点的集群里
kubeadm join 192.168.1.20:6443 --token hbovty.6x82bkdlsk6dfy32 \
    --discovery-token-ca-cert-hash sha256:659511b431f276b2a5f47397677b1dff74838ae5eb18e24135e6dae1b8c45840
```

### 验证集群

```shell
kubectl get nodes
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506190815.png)

## 安装 tomcat

1. 运行 deployment

```shell
kubectl create deployment my-tomcat --image=tomcat:7.0.75-alpine
```

2. 暴露 service

```shell
kubectl expose deployment my-tomcat --name=tomcat --port=8080 --type=NodePort
```

3. 验证, 浏览器输入http://192.168.1.21:31520 查看

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506192139.png)

## 其他操作命令

1. 创建部署

```shell
kubectl create deployment my-tomcat --image=tomcat:7.0.75-alpine
```

2. 查看pod 日志

```shell
kubectl logs my-tomcat-685b8fd9c9-rw42d(pod名称)
```

3. 进入Pod 容器

```shell
kubectl exec my-tomcat-685b8fd9c9-rw42d -- sh
```

4. 暴露服务

```shell
kubectl expose deployment my-tomcat --name=tomcat --port=8080 --type=NodePort
```

5. 删除 pod,service,deployment

```shell
kubectl delete pod pod-name -n [namespace]
kubectl delete service service-name -n [namespace]
kubectl delete deployment deplotment-name -n [namespace]
```

6. 扩容

```shell
kubectl scale --replicas=5 deployment my-tomcat
```

7. 查看 Pod 详细信息

```shell
kubectl describe pod my-tomcat-547db86547-4btmd
```

8. 回滚

```shell
# 查看历史版本
kubectl rollout history deploy my-tomcat 
# 回滚到上一个版本
kubectl rollout undo deployment my-tomcat     #--to-revision 参数可以指定回退的版本
```


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/8a64657/  

