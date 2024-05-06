# Centos ssh 免密登录配置


## ssh 免密登录配置
1. 生成公钥秘钥对
```shell
ssh-keygen -t rsa -b 4096 -C &#34;telzhou618@qq.com&#34;
```
在文件在 ~/.ssh 文件夹下，会生成两个文件id_rsa是私钥，id_rsa.pub是公钥。
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502131951.png)
2. 使用ssh复制公钥到目标服务器，会存储在目标服务的 authorized_keys文件中，下次登录目标服务不再需要密码。
```shell
ssh-copy-id -i ~/.ssh/id_rsa.pub  root@192.168.1.6
```
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240502132307.png)


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/f460746/  

