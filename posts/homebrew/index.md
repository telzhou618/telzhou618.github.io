# Homebrew 常用命令

## mac 软件包管理工具
```shell
brew install  #[包名] 安装包。
brew uninstall #[包名] 卸载包。
brew search #[包名] 查询包。
brew list   #列出已安装的软件。
brew update #更新brew。 -
brew outdated #查看可更新的包。
brew upgrade #(更新所有) 更新所有包。
brew upgrade [包名] #更新置顶包。
brew cleanup #清理所有包的旧版本。
brew cleanup [包名] #清理指定包的旧版本。
brew cleanup -n #查看可清理的旧版本包，不执行实际操作。
brew home #用浏览器打开brew的官方网站。
brew info [包名] #显示软件信息。
brew deps [包名] #显示包依赖。
brew install --cask [包名] #安装桌面软件。
```

## 服务管理
```shell
brew services list  # 查询服务列表和状态
brew services run mysql # 启动mysql 服务
brew services start mysql # 启动 mysql 服务，并注册开机自启
brew services stop mysql # 停止 mysql 服务，并取消开机自启
brew services restart mysql # 重启 mysql 服务，并注册开机自启
brew services cleanup # 清除已卸载应用的无用配置
```


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/homebrew/  

