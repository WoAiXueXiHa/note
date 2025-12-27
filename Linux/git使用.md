# 1. 基本操作

## 本地仓库相关操作

### 新建本地仓库

**仓库是进行版本控制的一个文件目录**。要对文件进行版本控制，必须先创建仓库

**创建仓库命令`git init`**，需要注意：要在文件目录下执行：

```bash
[vect@VM-0-11-centos git_show]$ pwd
/home/vect/git_repo/git_show
[vect@VM-0-11-centos git_show]$ git init
Initialized empty Git repository in /home/vect/git_repo/git_show/.git/
[vect@VM-0-11-centos git_show]$ ls -a
.  ..  .git

```

当前目录下多了一个`.git`隐藏文件，**`.git`目录是git来跟踪管理仓库的，不要手动修改这个目录里的任何文件！！！**

### 配置本地仓库

创建完本地仓库后就要设置**用户名称**和**e-mail地址**，配置命令：

`git config [--global] user.name"name"`

`git config [--global] user.email"email"`

`--global`是一个可选项，表示这台机器上所有git仓库都会使用这个配置

**查看配置命令：** `git config -l`

**删除配置命令：**

`git config [--global] --unset user.name`

`git config [--global] --unset user.email`

### 认识工作区、暂存区、版本库

> * **工作区：** 电脑上你要写代码或文件的目录
> * **暂存区(stage/index)：** 存放在`.git`目录下的`index`文件中
> * **版本库(repository):** 在工作区发现的隐藏目录`.git`不属于工作区，而是git的版本库，这个版本库里的所有文件都可以被git管理起来

再次强调：**不要改动`.git`目录！！！**

![image-20251225112338337](https://gitee.com/binary-whispers/pic/raw/master///20251225112340777.png)

* 在创建版本库时，git会自动创建一个唯一的`main`分支，以及指向`main`的指针`HEAD`
* 对工作区进行修改的文件执行`git add`命令时，暂存区目录树的文件索引会被更新
* 执行提交操作`git commit`命令时，`main`分支会更新，可以理解为暂存区的目录树此时才真正被写入到版本库里

由此可知：

**通过新建或粘贴进目录的文件，并不能称为向仓库中新增文件，而知识在工作区里新增了文件。必须经过`git add`和`git commit`命令才能将文件添加到仓库进行管理！！！**

## 添加文件

在包含`.git`的目录下新建文件，使用`git add`命令可以将文件添加到暂存区：

> * `git add [file1] [file2] ...`：添加一个或多个文件到暂存区
> * `git add [dir]`:添加指定目录到暂存区，包括子目录
> * `git add .`：添加当前目录下所有改动文件到暂存区，这个操作用的最多

再使用`git commit`命令将暂存区内容添加到本地仓库

> * `git commit -m "描述内容"`提交暂存区全部内容到本地仓库
> * `git commit [file1] [file2] ... -m "描述内容"`：提交暂存区指定文件到仓库

```bash
[vect@VM-0-11-centos git_show]$ ls
ReadMe
[vect@VM-0-11-centos git_show]$ git add .
[vect@VM-0-11-centos git_show]$ git commit -m "add ReadMe"
[master (root-commit) 77326a4] add ReadMe
 1 file changed, 2 insertions(+)
 create mode 100644 ReadMe

```

我们可以用`git log --pretty=oneline`（后面的选项是为了显示更简洁的信息）命令查看历史提交记录：

```bash
[vect@VM-0-11-centos git_show]$ git log --pretty=oneline
77326a49381d63607c026271b665f5441937bb82 add ReadMe

```

`77326a49381d63607c026271b665f5441937bb82 `这串字符是git的**commit id**，这是哈希计算出来的编码，十六进制表示

我们现在查看一下`.git`目录结构：

```bash
[vect@VM-0-11-centos git_show]$ tree .git
.git
|-- branches
|-- COMMIT_EDITMSG
|-- config
|-- description
|-- HEAD
|-- hooks
|   |-- applypatch-msg.sample
|   |-- commit-msg.sample
|   |-- post-update.sample
|   |-- pre-applypatch.sample
|   |-- pre-commit.sample
|   |-- prepare-commit-msg.sample
|   |-- pre-push.sample
|   |-- pre-rebase.sample
|   `-- update.sample
|-- index
|-- info
|   `-- exclude
|-- logs
|   |-- HEAD
|   `-- refs
|       `-- heads
|           `-- master
|-- objects
|   |-- 77
|   |   `-- 326a49381d63607c026271b665f5441937bb82
|   |-- 9c
|   |   `-- b300e6c96804e867310ab2084ff954b788baa5
|   |-- b1
|   |   `-- d63518a51fcbd65e3ba73823674429f8f57105
|   |-- info
|   `-- pack
`-- refs
    |-- heads
    |   `-- master
    `-- tags

15 directories, 21 files

```

> * `index`就是暂存区，add后的内容都添加到这里
>
> * `HEAD`默认指向master分支：
>
>   ```bash
>   [vect@VM-0-11-centos git_show]$ cat .git/HEAD
>   ref: refs/heads/master
>   ```
>
>   注意：我前面画的main就是master分支，GitHub显示的是main，二者没有区别
>
> * 默认的master分支
>
>   ```bash
>   [vect@VM-0-11-centos git_show]$ cat .git/refs/heads/master 
>   77326a49381d63607c026271b665f5441937bb82
>   ```
>
>   这串哈希字符就是当前最新的**commit id**
>
> * `objects`是git的对象库，包含了创建的各种版本库对象和内容。执行`git add`时，暂存区的目录树被更新，工作区修改的文件被写入到对象库中一个新的对象中，位于`.git/objects`目录下：
>
>   ```bash
>   [vect@VM-0-11-centos git_show]$ ls .git/objects/
>   77  9c  b1  info  pack
>   ```
>
>   **commit id**的前2位是文件夹名称，后38位是文件名称
>
>   可以用`git cat-file`产看版本库对象内容：
>
> ```bash
> [vect@VM-0-11-centos git_show]$ git log --pretty=oneline
> 4beaba88b98e2bde00d6fd19f0a266cbf9ee7f88 add a line
> 77326a49381d63607c026271b665f5441937bb82 add ReadMe
> 
> [vect@VM-0-11-centos git_show]$ git cat-file -p 4beaba88b98e2bde00d6fd19f0a266cbf9ee7f88
> tree 5d5d317da2a20b2c90a89360b77c0dba21a4ae68
> parent 77326a49381d63607c026271b665f5441937bb82
> author WoAiXueXiHa <1760198676@qq.com> 1766729302 +0800
> committer WoAiXueXiHa <1760198676@qq.com> 1766729302 +0800
> 
> add a line
> # 这就是最近更新的所有文件
> ```
>
> 我们再看这一行`tree 5d5d317da2a20b2c90a89360b77c0dba21a4ae68`
>
> ```bash
> [vect@VM-0-11-centos git_show]$ git cat-file -p 5d5d317da2a20b2c90a89360b77c0dba21a4ae68
> 100644 blob 84309d87a0487c0648ea7603da9af242ff487c76	ReadMe
> [vect@VM-0-11-centos git_show]$ git cat-file -p 84309d87a0487c0648ea7603da9af242ff487c76
> # 这是git操作演示
> # 用于git操作练习和教学
> # 再添加一行内容
> [vect@VM-0-11-centos git_show]$ 
> 
> ```
>
> 我们就可以找到`ReadMe`这个文件的详细内容！

总结一下，`.git`目录下有几个目录文件很关键：

* `index`：暂存区，`git add`后会更新内容
* `HEAD`： 指针，默认指向`master`分支
* `ref/heads/master`：文件里保存当前`master`分支最新的`commit id`
* `objects`：包含创建各种版本库对象及内容，可以理解为存放了git维护的所有修改内容

## 修改文件

**git跟踪并管理的是修改，而不是文件**

使用`git status`查看当前仓库状态

```bash
[vect@VM-0-11-centos git_show]$ git status
# On branch master
nothing to commit, working directory clean
[vect@VM-0-11-centos git_show]$ vim ReadMe
[vect@VM-0-11-centos git_show]$ git status
# On branch master
# Changes not staged for commit:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#	modified:   ReadMe
#
no changes added to commit (use "git add" and/or "git commit -a")
```

使用`git diff [file]`显示暂存区和工作区的文件差异

```bash
[vect@VM-0-11-centos git_show]$ vim ReadMe
[vect@VM-0-11-centos git_show]$ git diff ReadMe
diff --git a/ReadMe b/ReadMe
index 7ce7c0c..0be418d 100644
--- a/ReadMe
+++ b/ReadMe
@@ -2,4 +2,4 @@
 # 用于git操作练习和教学
 # 再添加一行内容
 # 这一行是新修改的内容 演示 git status
-
+# 这一行用于演示 git diff

```

## 版本回退

本质是回退版本库的内容，对于工作区和暂存区请看如下表格：

`git reset [--soft] [--mixed] [--hard] [HEAD]`

![image-20251226144104494](https://gitee.com/binary-whispers/pic/raw/master///20251226144107076.png)

* `--soft`：只回退版本库内容
* `--mixed`：回退版本库和暂存区，默认选项，使用时可以不带这个选项
* `--hard`：三个区域全部回退，慎用
* `HEAD`说明：
  * 可以直接写`commit id`，指定回退的版本
  * `HEAD`是当前版本
  * `HEAD^`上一个版本
  * `HEAD^^`上上一个版本
  * ....

现在准备三个版本的`ReadMe`文件：

```bash
[vect@VM-0-11-centos git_show]$ vim ReadMe
[vect@VM-0-11-centos git_show]$ git add .
[vect@VM-0-11-centos git_show]$ git commit -m "version1"
[master 7eab523] version1
 1 file changed, 3 insertions(+)
[vect@VM-0-11-centos git_show]$ vim ReadMe
[vect@VM-0-11-centos git_show]$ git add .
[vect@VM-0-11-centos git_show]$ git commit -m "version2"
[master 67c21b9] version2
 1 file changed, 1 insertion(+)
[vect@VM-0-11-centos git_show]$ vim ReadMe
[vect@VM-0-11-centos git_show]$ git add .
[vect@VM-0-11-centos git_show]$ git commit -m "version3"
[master 9069769] version3
 1 file changed, 1 insertion(+)
[vect@VM-0-11-centos git_show]$ cat ReadMe
# 这是git操作演示
# 用于git操作练习和教学
# 再添加一行内容
# 这一行是新修改的内容 演示 git status
# 这一行用于演示 git diff
# 版本一
# 版本二
# 版本三
```

所有的修改操作：

```bash
[vect@VM-0-11-centos git_show]$ git log --pretty=oneline
9069769cabf518eb50451dbda267bb5c855a86c9 version3
67c21b917930b5c0ff8507a51119e0fbdebd60f3 version2
7eab52360e77a28a3a97c1bb37ec62fddb3cb333 version1
4beaba88b98e2bde00d6fd19f0a266cbf9ee7f88 add a line
77326a49381d63607c026271b665f5441937bb82 add ReadMe
```

想把所有内容都回退到版本二：

```bash
[vect@VM-0-11-centos git_show]$ git reset --hard HEAD^
HEAD is now at 67c21b9 version2
[vect@VM-0-11-centos git_show]$ git log --pretty=oneline
67c21b917930b5c0ff8507a51119e0fbdebd60f3 version2
7eab52360e77a28a3a97c1bb37ec62fddb3cb333 version1
4beaba88b98e2bde00d6fd19f0a266cbf9ee7f88 add a line
77326a49381d63607c026271b665f5441937bb82 add ReadMe
[vect@VM-0-11-centos git_show]$ cat ReadMe
# 这是git操作演示
# 用于git操作练习和教学
# 再添加一行内容
# 这一行是新修改的内容 演示 git status
# 这一行用于演示 git diff
# 版本一
# 版本二

```

撤销回退版本操作：`git reset --hard 对应版本的commit id`

```bash
[vect@VM-0-11-centos git_show]$ git reset --hard 9069769cabf518eb50451dbda267bb5c855a86c9
HEAD is now at 9069769 version3
[vect@VM-0-11-centos git_show]$ git log --pretty=oneline
9069769cabf518eb50451dbda267bb5c855a86c9 version3
67c21b917930b5c0ff8507a51119e0fbdebd60f3 version2
7eab52360e77a28a3a97c1bb37ec62fddb3cb333 version1
4beaba88b98e2bde00d6fd19f0a266cbf9ee7f88 add a line
77326a49381d63607c026271b665f5441937bb82 add ReadMe
[vect@VM-0-11-centos git_show]$ cat ReadMe
# 这是git操作演示
# 用于git操作练习和教学
# 再添加一行内容
# 这一行是新修改的内容 演示 git status
# 这一行用于演示 git diff
# 版本一
# 版本二
# 版本三

```

如果往上翻不到对应版本的`commit id`，使用`git reflog`查询每次命令：

```bash
[vect@VM-0-11-centos git_show]$ git reflog
9069769 HEAD@{0}: reset: moving to 9069769cabf518eb50451dbda267bb5c855a86c9
67c21b9 HEAD@{1}: reset: moving to HEAD^
9069769 HEAD@{2}: commit: version3
67c21b9 HEAD@{3}: commit: version2
7eab523 HEAD@{4}: commit: version1
4beaba8 HEAD@{5}: commit: add a line
77326a4 HEAD@{6}: commit (initial): add ReadMe

```

![image-20251226152655092](https://gitee.com/binary-whispers/pic/raw/master///20251226152658972.png)

切换版本的速度极快，是因为只用改变指针指向即可

## 撤销修改

### 情况一：工作区的代码还没有`add`

`git checkout -- [file]`让工作区的文件回到最近一次`add`或`commit`的状态，注意`--`一定不能省略，否则意思就变了

```bash
[vect@VM-0-11-centos git_show]$ cat ReadMe
# 这是git操作演示
# 用于git操作练习和教学
# 再添加一行内容
# 这一行是新修改的内容 演示 git status
# 这一行用于演示 git diff
# 版本一
# 版本二
# 版本三
# 这一行用于演示 git checkout -- file
[vect@VM-0-11-centos git_show]$ git checkout -- ReadMe
[vect@VM-0-11-centos git_show]$ cat ReadMe
# 这是git操作演示
# 用于git操作练习和教学
# 再添加一行内容
# 这一行是新修改的内容 演示 git status
# 这一行用于演示 git diff
# 版本一
# 版本二
# 版本三

```

### 情况二：已经`add`但没有`commit`

`git`
