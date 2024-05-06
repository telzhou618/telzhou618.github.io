# centos 网络错误问题解决


&lt;!--more--&gt;

虚拟机 centos 启动后网络无法启动报错

# 解决报错Failed to start LSB: Bring up/down networking

一般情况是网络冲突了或者mac地址冲突。

查看网络状态

```shell
systemctl status network
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506220832.png)

## 解决方案1

禁用 NetworkManager

```shell
 systemctl stop NetworkManager
 systemctl disable NetworkManager
```

重启网络服务

```shell
systemctl start network
```

如果还没好, 尝试以下家解决方案

## 解决方案2

查看MAC地址

```shell
ip a
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506221215.png)

修改网络配置文件, 加上 mac 地址

```shell
cd /etc/sysconfig/network-scripts/
vim ifcfg-ens33
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506221315.png)

最后重启网络看看！

---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/57808a3/  

