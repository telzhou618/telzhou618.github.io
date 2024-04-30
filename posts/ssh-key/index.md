# SSK key 免密访问 gitlab 配置


## 安装最新 git
安装在最新yum源,ius源官方：https://ius.io/setup
```shell
yum install \
https://repo.ius.io/ius-release-el7.rpm \
https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```
安装 git
```shell
 yum remove git
 yum install -y git236
 git --version

```

## ssh-gen 命令
举个例子，生成算法为rsa，长度为4096字节的秘钥对。
```shell
ssh-keygen -t rsa -b 4096 -C &#34;telzhou618@qq.com&#34;
```
参数说明：
![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240501011832.png)

## linux 生成本地秘钥
生成 rsa 算法公钥和秘钥,运行ssk-keygen 一路回车，注意，如果秘钥已存在会覆盖
```shell
ssh-keygen -t rsa -b 4096 -C &#34;telzhou618@qq.com&#34;
```
- -t ：指定算法类型
- -b ：指定秘钥长度
- -C ：大C，注释说明
- -f ：保存秘钥的文件名

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240423100359.png)

复制公钥（id_rsa.pub）文件中的内容填写在gitlab中。

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240423095344.png)

然后再 git clone, 不需要再输入用户名和密码。

## windows 配置

生成秘钥方法和Linux类似
```shell
ssh-keygen -t rsa -b 4096 -C &#34;telzhou618@qq.com&#34;
cat /c/Users/Administrator/.ssh/id_rsa.pub
```

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240423100106.png)

设置到 gitlab, 和前面linux一样。

## 本地多秘钥生成
比如要同时免密访问github和gitlab仓库，就需要生成两组秘钥，入校所示：

1. 为gitlab生成秘钥,-f指定文件名称
```shell
ssh-keygen -t rsa -C &#39;yourEmail@xx.com&#39; -f ~/.ssh/gitlab-rsa
```
2. 为github生成秘钥
```shell
ssh-keygen -t rsa -C &#39;yourEmail2@xx.com&#39; -f ~/.ssh/github-rsa
```
3. 在~/.ssh目录下创建名为config的文件，用于配置不同host的不同ssk key。
```shell
touch ~/.ssh/config
```
内容如下所示
```shell
# gitlab
Host gitlab.com
    HostName gitlab.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/gitlab_id-rsa
# github
Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/github_id-rsa
    # 配置文件参数
    # Host : Host可以看作是一个你要识别的模式，对识别的模式，进行配置对应的的主机名和ssh文件
    # HostName : 要登录主机的主机名
    # User : 登录名
    # IdentityFile : 指明上面User对应的identityFile路径
```
4. 分别在github 和 gitlab 配置公钥信息, 这样就可以再不同平台的代码仓库免密克隆项目了。




---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/ssh-key/  

