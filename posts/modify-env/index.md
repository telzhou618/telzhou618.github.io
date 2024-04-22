# Mac 修改环境变量

## 环境变量文件存储位置
```markdown
&gt; /etc/paths （全局建议修改这个文件 ）
&gt; /etc/profile （建议不修改这个文件 ）
&gt; /etc/bashrc （一般在这个文件中添加系统级环境变量）
&gt; ~/.profile 文件为系统的每个用户设置环境信息,当用户第一次登录时,该文件被执行.并从/etc/profile.d目录的配置文件中搜集shell的设置
&gt; ~/.bashrc 每一个运行bash shell的用户执行此文件.当bash shell被打开时,该文件被读取
&gt; ~/.bash_profile 该文件包含专用于你的bash shell的bash信息,当登录时以及每次打开新的shell时,该文件被读取
```

## 修改完让其生效
```shell
source ~/.bash_profile
```   


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/modify-env/  

