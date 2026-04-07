# 创建本地仓库

在一个目录下创建：`git init`

**为仓库新增两个配置项目**

`git config user.name "vect"`

`git config user.email "123@qq.com"`

查看配置：

`git config -l`

删除配置：

`git config --unset user.name`



全局，本地仓库里所有的配置都是这个配置项

`git config --global user.name "..."`

`git config --global user.email "..."`

删除全局配置：

`git config --global --unset`



# 认识工作区 暂存区 版本库

**不允许在`.git/`下手动修改任何东西！**

![image-20260331213440797](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260331213440797.png)

## 添加文件

`git add .`当前目录下所有修改的文件放入暂存区

`git commit -m "描述"`本次提交的细节-->版本变更信息

`git log`查看提交日志

`git log --pretty=oneline`



## 修改文件

git追踪管理的是修改，而不是文件

对象库中的对象存的是修改的内容

`git status`查看仓库状态，只能知道是否改了没

`git diff 文件名`查看发生哪些变化

 

## 版本回退

`git reset [--soft | --mixed | --hard] [commitID]`

按照强度，左到右强度依次增大

回退版本库

回退版本库和暂存区

全部回退

**撤销回退，反悔：**

`git reset commitID`

找不到commitID咋办？

`git reflog` 有所有的改动commitID信息

**版本回退为什么这么快？修改指针指向**

HEAD指针指向main，main指向版本

![image-20260331215637514](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260331215637514.png)



## 撤销修改

### 撤销工作区的修改---还没有add呢

`git checkout -- 文件名` `--`很重要，没有就改变意思了

## 撤销暂存区的修改--还没commit呢

法一：

1. `git reset -mixed(默认选项，可省略) HEAD^`，有几个`^`就往前退几个版本----->这时候就是只撤销了暂存区的修改，如果要撤销工作区的修改，继续
2. `git checkout -- 文件名`

法二：

直接`git reset -hard`，工作区的修改也撤销了

## 撤销版本库的修改：前提--commit之后没有push呢

撤销的目的就是不影响远程仓库代码

`git reset 选项自己选就好` 



## 删除文件

先删除工作区，然后 add commit 三步走

`git rm 文件` 删除工作区+暂存区 `git commit` 版本库删除 两步走



# 分支管理

## 理解分支管理

![image-20260331221952543](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260331221952543.png)



