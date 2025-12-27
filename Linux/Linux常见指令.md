# 一、增：新建目录和文件

在操作之前先要理解Linux的**文件系统的树状目录：**所有文件和目录以根目录`/`为起点，形成层级树状结构，普通文件和空目录是叶子节点，目录是分支节点（包含子目录或文件）

```bash
[root@VM-0-11-centos ~]# tree -L 1 /
/
|-- bin -> usr/bin
|-- boot
|-- data
|-- dev
|-- etc
|-- home
|-- lib -> usr/lib
|-- lib64 -> usr/lib64
|-- lost+found
|-- media
|-- mnt
|-- opt
|-- proc
|-- root
|-- run
|-- sbin -> usr/sbin
|-- srv
|-- sys
|-- tmp
|-- usr
`-- var

21 directories, 0 files
[root@VM-0-11-centos ~]# 
```

用这个方式就可以查看Linux根目录下的一级目录，如下图所示：

![目录树](https://gitee.com/binary-whispers/pic/raw/master///20251204193645203.png)

## 1. 创建目录：mkdir

**目录的递归创建：**若需要创建多级嵌套目录(如“`a/b/c`)直接使用`mkdir a/b/c`会报错（因父目录`a`和`a/b`不存在），需要通过`-p`选项自动创建所有缺失的父目录。

**语法和选项**

>  `mkdir [选项] 目录名...`

| **选项** | **说明**         |
| -------- | ---------------- |
| `-p`     | 递归创建多级目录 |
| 无选项   | 仅创建单级空目录 |

**案例**

```bash
# 1. 创建单级目录
[root@VM-0-11-centos ~]# mkdir path1
[root@VM-0-11-centos ~]# ls
path1

# 2. 递归创建多级目录
[root@VM-0-11-centos ~]# mkdir -p path1/path2/path3
[root@VM-0-11-centos ~]# tree
.
`-- path1
    `-- path2
        `-- path3

3 directories, 0 files
```

## 2.创建文件：touch

**概念铺垫：文件 = 文件属性 + 文件内容**

这是一个文件属性的具体信息：

![文件属性](https://gitee.com/binary-whispers/pic/raw/master///20251204193700992.png)

对于**文件的时间属性：**Linux有三个关键时间戳，`touch`指令的核心作用是修改这些时间戳，如果文件不存在自动创建空文件

**时间戳：**是一种**时间存储标准**，指从「1970 年 1 月 1 日 00:00:00 UTC」（Unix 纪元时间）到当前时间的「总秒数」（不含闰秒），比如 `1730000000` 代表某个具体时间。

```bash
[root@VM-0-11-centos ~]# stat path1
  File: ‘path1’
  Size: 4096      	Blocks: 8          IO Block: 4096   directory
Device: fd01h/64769d	Inode: 919083      Links: 3
Access: (0755/drwxr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2025-11-30 15:56:58.553850382 +0800			# 访问时间
Modify: 2025-11-30 15:56:56.327782460 +0800			# 修改时间
Change: 2025-11-30 15:56:56.327782460 +0800			# 变化时间
 Birth: -

```

**语法：**

> `touch [选项] 文件名...`

| **选项** | **说明**                                                     |
| -------- | ------------------------------------------------------------ |
| `-a`     | 仅修改文件的访问时间（atime）                                |
| `-m`     | 仅修改文件的修改时间（mtime）                                |
| 无选项   | 若文件不存在则创建空文件，若存在则更新atime和mtime为当前时间 |

```bash
[root@VM-0-11-centos path1]# touch code.c
[root@VM-0-11-centos path1]# ls -l
total 4
-rw-r--r-- 1 root root    0 Nov 30 16:15 code.c
-rw-r--r-- 1 root root    0 Nov 30 16:14 code.txt
drwxr-xr-x 3 root root 4096 Nov 30 15:56 path2
[root@VM-0-11-centos path1]# touch code.txt
[root@VM-0-11-centos path1]# ls -l
total 4
-rw-r--r-- 1 root root    0 Nov 30 16:15 code.c
-rw-r--r-- 1 root root    0 Nov 30 16:15 code.txt
drwxr-xr-x 3 root root 4096 Nov 30 15:56 path2
```

## 3. 文件写入与读取：echo printf cat tac

这里需要一个新的认知：**Linux下一切皆文件！！！**

类比C++：

> `cout`：在显示器上打印文本，即向显示器文件写入数据
>
> `cin`：键盘接收输入数据，即从键盘文件读取数据

而**重定向：**

1. 输出重定向：`>`表示**覆盖写入**，`>>`表示**追加写入**
2. 输入重定向：`<`表示**读文件当输入**

**案例：（请看详细注释）**

```bash
# 向显示器写入文本
[root@VM-0-11-centos path1]# echo "123456"
123456
# echo和printf的唯一区别就是一个有换行一个没有
[root@VM-0-11-centos path1]# printf "132456"
132456[root@VM-0-11-centos path1]# 
# >输出重定向 echo原本要将"hello Linux"输出到显示屏文件 但是写入到了文件中
# 若文件存在 则删除原先内容 从头开始写入
# 若文件不存在 则新建文件并写入
132456[root@VM-0-11-centos path1]# tree
.
|-- code.c
|-- code.txt
`-- path2
    `-- path3

2 directories, 2 files
[root@VM-0-11-centos path1]# pwd
/root/path1
[root@VM-0-11-centos path1]# echo "hello Linux" > code.txt
[root@VM-0-11-centos path1]# tree
.
|-- code.c
|-- code.txt
`-- path2
    `-- path3

2 directories, 2 files
[root@VM-0-11-centos path1]# cat code.txt
hello Linux

# 原文件存在 删除原来的内容 从头写入
[root@VM-0-11-centos path1]# echo "123456" > code.txt
[root@VM-0-11-centos path1]# cat code.txt
123456
# 追加写入
[root@VM-0-11-centos path1]# echo "7897978798" >> code.txt
[root@VM-0-11-centos path1]# cat code.txt
123456
7897978798
# 带行号显示文件内容
[root@VM-0-11-centos path1]# cat -n code.txt
     1	123456
     2	7897978798
# 逆序显示
[root@VM-0-11-centos path1]# tac code.txt
7897978798
123456

[root@VM-0-11-centos path1]# > code.txt
[root@VM-0-11-centos path1]# cat code.txt
[root@VM-0-11-centos path1]# > show.txt
[root@VM-0-11-centos path1]# tree
.
|-- code.c
|-- code.txt
|-- path2
|   `-- path3
`-- show.txt

2 directories, 3 files
[root@VM-0-11-centos path1]# 

```

这里就可以总结出`>`新的用法：

> * 若原文件存在，清空文件
> * 若原文件不存在，新建文件

****

# 二、 删：删除目录和文件

## 1. rm

这里需要注意：Linux系统并没有回收站，文件删除就真的找不到了，所以删除文件之前要三思

* **文件与目录的删除差异**：`rm`默认仅能删除文件，删除目录需加`-r`选项（递归删除目录内所有内容）；若需强制删除（跳过确认提示），需加`-f`选项（强制删除，忽略不存在的文件和权限提示）。
* **安全提示**：`rm -rf /`是极其危险的指令（会删除根目录下所有文件，导致系统崩溃），绝对禁止使用。

**语法：**`rm [选项] 文件名/目录名...`

| **选项** | **说明**                                   |
| -------- | ------------------------------------------ |
| `-r`     | 递归删除目录（包含目录内所有文件和子目录） |
| `-f`     | 强制删除（不做任何提示）                   |

**案例：**

```bash
[root@VM-0-11-centos path1]# tree
.
|-- code.c
|-- code.txt
|-- path2
|   `-- path3
`-- show.txt

2 directories, 3 files
# 删除单个文件 会有提示
[root@VM-0-11-centos path1]# rm code.c
rm: remove regular empty file ‘code.c’? y
[root@VM-0-11-centos path1]# ls
code.txt  path2  show.txt
# 强制删除单个文件
[root@VM-0-11-centos path1]# rm -f show.txt
[root@VM-0-11-centos path1]# ls
code.txt  path2
# 强制递归删除目录
[root@VM-0-11-centos path1]# rm -rf path2
[root@VM-0-11-centos path1]# ls
code.txt
```

## 2. rmdir

仅删除**空目录**，没有`rm -r`灵活

**语法： `rmdir [选项] 目录名...`**

```bash
[root@VM-0-11-centos ~]# mkdir -p show1/show2/show3
[root@VM-0-11-centos ~]# tree
.
|-- path1
|   `-- code.txt
`-- show1
    `-- show2
        `-- show3

4 directories, 1 file
[root@VM-0-11-centos ~]# cd /root/show1/show2
[root@VM-0-11-centos show2]# pwd
/root/show1/show2
[root@VM-0-11-centos show2]# > code.txt | > code.c
[root@VM-0-11-centos show2]# ls
code.c  code.txt  show3

[root@VM-0-11-centos ~]# rmdir -p show1/show2/show3
# 很鸡肋 别用
rmdir: failed to remove directory ‘show1/show2’: Directory not empty
[root@VM-0-11-centos ~]# 

```



****

# 三、 查：获取文件/目录信息

这个操作涵盖了**查看目录内容、文件内容、文件属性、系统信息**等场景，需要结合**路径概念**和**管道操作**

## 1.相关概念

### 家目录

家目录是 Linux 为每个用户分配的「专属私人文件夹」，核心作用是让用户存放自己的文件、配置和数据，且每个用户默认只能修改自己家目录的内容（保证隐私和安全）。

> * **普通用户**：家目录统一放在 `/home` 目录下，名字和用户名一致，比如用户 `whb` 的家目录是 `/home/whb`。
> * **root 用户（超级管理员）**：家目录单独在 `/root`（不跟普通用户混放，因为权限最高）。
> * `cd ~`：快速进入家目录

###  路径：定位文件/目录的唯一方式

> * **绝对路径：**从根目录`/`开始的的完整路径，如：`/root/path1/code.txt`，唯一性强，不受当前目录的影响，一般用于配置文件和脚本
> * **相对路径：**相对于当前所处位置的路径（比如说我当前在`/root/path1`,则`/path1/code.txt`等价于`/root/path1/code.txt`），便捷性高，常用于命令行交互
> * **特殊符号：** `./`当前目录 `../`上级目录 `~`当前用户的家目录（如`~`对普通用户是`/home/用户名`，对 root 是`/root`）

**为什么路径是定位文件/目录的唯一方式？**

![路径唯一](https://gitee.com/binary-whispers/pic/raw/master///20251204193708313.png)

对于多叉树：

> * **任何一个叶子节点有且仅有一个父节点**
> * **任何一个节点存在0个或多个子节点**

我们从`/local`开始，一路找父节点：`local->/user->/`

而从根目录`/`开始，顺着这个条路径走：`/->/user->local`

会发现**路径是唯一的**，所以可以定位一个文件

### 管道操作`|`

`|`这也是文件，将前一个命令作为后一个命令的输入

## 2. ls 查看目录内容

**概念**

* **文件属性解析：** `ls -l`显示文件的详细信息

![image-20251205193621442](https://gitee.com/binary-whispers/pic/raw/master///20251205193625348.png)

> 1. 字段首位：`d`代表目录 `-`代表普通文件
> 2. 权限（`rwxr-xr-x`）:每三个为一组，分别属于拥有者u、所属组g、其他用户o）每组`r`为读权限、`w`为写权限、`x`为执行权限
> 3. 链接数（`3`）：文件的硬链接数量
> 4. 拥有者（`root`）：文件的归属用户
> 5. 所属组（`root`）：文件归属的用户组，这里就可以理解一个项目由一个项目组负责，这个项目属于项目组的所有用户
> 6. 大小（`4096`）：文件大小，单位：字节
> 7. 修改时间（`Nov 30 15:56`）：文件的最后修改时间
> 8. 文件名（`path1`）：文件或目录名称

**语法：`ls [选项] [目录/文件]`**

| **常用选项** | **说明**                                      |
| ------------ | --------------------------------------------- |
| `-l`         | 长格式显示，包含文件详细属性                  |
| `-a`         | 显示所有文件，包含隐藏文件（以`.`开头的文件） |
| `-d`         | 仅显示目录本身，不显示目录内的内容            |
| `-t`         | 按修改时间排序，最新修改的在最前面            |
| `-R`         | 递归显示目录内容                              |

这些选项可以组合使用

```bash
[root@VM-0-11-centos path1]# pwd
/root/path1
[root@VM-0-11-centos path1]# ls
code.txt
[root@VM-0-11-centos path1]# ls -al
total 8
drwxr-xr-x   2 root root 4096 Nov 30 16:57 .
dr-xr-x---. 10 root root 4096 Nov 30 17:03 ..
-rw-r--r--   1 root root    0 Nov 30 16:49 code.txt
[root@VM-0-11-centos path1]# ls -alt
total 8
dr-xr-x---. 10 root root 4096 Nov 30 17:03 ..
drwxr-xr-x   2 root root 4096 Nov 30 16:57 .
-rw-r--r--   1 root root    0 Nov 30 16:49 code.txt
```

## 3. 查看文件内容： cat/more/less/head/tail

**文件查看工具的核心差异：**

* `cat`：一次性显示所有内容（适合小文件，大文件会刷屏）
* `more/less`：分页显示（适合大文件），`more`只支持向下翻页，`less`支持上下翻页和搜索
* `head/tail`：显示文件开头/结尾（默认10行，可指定行数）

**各种指令语法和选项**

| **指令** | **语法**             | **核心选项**                                                 | **功能说明**                       |
| -------- | -------------------- | ------------------------------------------------------------ | ---------------------------------- |
| `cat`    | `cat [选项] [文件]`  | `-n`:带行号显示 `-b`: 仅显示非空行 `-s` ：把多个空行压缩成一个空行显示 | 一次性显示文件所有内容             |
| `more`   | `more [选项] [文件]` | `-n`：每页显示n行，这里的n是具体数字  `-q`：退出查看         | 分页显示，只能向下翻页，按空格翻页 |
| `less`   | `less [选项] [文件]` | `-N`：每页显示n行，这里的n是具体数字  `-q`：退出查看 `/字符串`：向下查找 `?字符串`：向上查找 | 分页显示，支持上下翻页、搜索       |
| `head`   | `head [选项] [文件]` | `-n`：显示前n行，这里的n是具体数字                           | 显示文件前n行                      |
| `tail`   | `tail [选项] [文件]` | `-n`：显示后n行，这里的n是具体数字 `-f`：实时跟踪文件更新    | 显示文件后n行，支持实时监控        |

**案例**

```bash
[root@VM-0-11-centos path1]# ls
1000lines.txt  10lines.txt  code.txt
# 按行号显示文件
[root@VM-0-11-centos path1]# cat -n 10lines.txt
     1	第1行：这是用for循环生成的内容
     2	第2行：这是用for循环生成的内容
     3	第3行：这是用for循环生成的内容
     4	第4行：这是用for循环生成的内容
     5	第5行：这是用for循环生成的内容
     6	第6行：这是用for循环生成的内容
     7	第7行：这是用for循环生成的内容
     8	第8行：这是用for循环生成的内容
     9	第9行：这是用for循环生成的内容
    10	第10行：这是用for循环生成的内容
# 逆序显示文件
[root@VM-0-11-centos path1]# tac 10lines.txt
第10行：这是用for循环生成的内容
第9行：这是用for循环生成的内容
第8行：这是用for循环生成的内容
第7行：这是用for循环生成的内容
第6行：这是用for循环生成的内容
第5行：这是用for循环生成的内容
第4行：这是用for循环生成的内容
第3行：这是用for循环生成的内容
第2行：这是用for循环生成的内容
第1行：这是用for循环生成的内容
# 每6行为一页显示 这里我按了两次空格 翻了两页
[root@VM-0-11-centos path1]# more -6 1000lines.txt
第1行：这是用while循环生成的内容
第2行：这是用while循环生成的内容
第3行：这是用while循环生成的内容
第4行：这是用while循环生成的内容
第5行：这是用while循环生成的内容
第6行：这是用while循环生成的内容
第7行：这是用while循环生成的内容
第8行：这是用while循环生成的内容
第9行：这是用while循环生成的内容
第10行：这是用while循环生成的内容
第11行：这是用while循环生成的内容
第12行：这是用while循环生成的内容
第13行：这是用while循环生成的内容
第14行：这是用while循环生成的内容
第15行：这是用while循环生成的内容
第16行：这是用while循环生成的内容
第17行：这是用while循环生成的内容
第18行：这是用while循环生成的内容
# 会跳转到新的页面显示 /字符串会向下查找 ?字符串会向上查找 q退出
[root@VM-0-11-centos path1]# less -6 1000lines.txt
# 显示[285,300]行 
# 也可以用这个公式 head -末尾行 文件名 | tail -$(末尾行 - 初始行 + 1)
[root@VM-0-11-centos path1]# head -300 1000lines.txt | tail -16
第285行：这是用while循环生成的内容
第286行：这是用while循环生成的内容
第287行：这是用while循环生成的内容
第288行：这是用while循环生成的内容
第289行：这是用while循环生成的内容
第290行：这是用while循环生成的内容
第291行：这是用while循环生成的内容
第292行：这是用while循环生成的内容
第293行：这是用while循环生成的内容
第294行：这是用while循环生成的内容
第295行：这是用while循环生成的内容
第296行：这是用while循环生成的内容
第297行：这是用while循环生成的内容
第298行：这是用while循环生成的内容
第299行：这是用while循环生成的内容
第300行：这是用while循环生成的内容
```

## 4. 辅助查询指令 find which

这里先引入`*`，这是一个通配符，用于**快速匹配一个或多个文件 / 目录名**的特殊符号，核心作用是 ***替代具体文件名的部分字符**，避免手动输入冗长的文件名，大幅提升操作效率

**`*`：匹配任意数量的任意字符（包括 0 个） **

* **作用**：最常用的通配符，代表 “任意长度的任意字符”（可以是字母、数字、符号，长度不限）。
* **语法**：`*` 可放在文件名的任意位置（开头、中间、结尾）。
* **实操案例**

```bash
[root@VM-0-11-centos path1]# tree
.
|-- 1000lines.txt
|-- 10lines.txt
|-- code.txt
|-- hello.c
`-- hello.png

0 directories, 5 files
# 显示所有以.txt为后缀的文件
[root@VM-0-11-centos path1]# ls *.txt
1000lines.txt  10lines.txt  code.txt
# 显示所有以hello开头的文件
[root@VM-0-11-centos path1]# ls hello*
hello.c  hello.png
# 显示所有包含o的文件
[root@VM-0-11-centos path1]# ls *o*
code.txt  hello.c  hello.png
# 删除所有以.c为后缀的文件
[root@VM-0-11-centos path1]# rm -rf *.c
[root@VM-0-11-centos path1]# tree
.
|-- 1000lines.txt
|-- 10lines.txt
|-- code.txt
`-- hello.png

0 directories, 4 files
```

**核心差异：**

* `find`：`find [路径] [选项]`按照文件名、文件大小、文件路径查找，可以递归遍历目录树
* `which`：`which [命令]`查找命令的路径

```bash
# 查找这个路径下的所有以.txt为后缀的文件
[root@VM-0-11-centos ~]# find /root/path1 "*.txt"
/root/path1
/root/path1/10lines.txt
/root/path1/code.txt
/root/path1/1000lines.txt
# 查找命令所在路径
[root@VM-0-11-centos ~]# which ls
alias ls='ls --color=auto'
	/usr/bin/ls
[root@VM-0-11-centos ~]# which ls -al
/usr/bin/which: invalid option -- 'l'
alias ls='ls --color=auto'
	/usr/bin/ls
```

# 四、改：修改文件/目录属性、内容与位置

修改涵盖修改文件内容、文件名/目录名、权限、所有者、位置等场景

## 1. mv （移动/重命名文件目录）

> * 重命名：当源和目标在同一目录时，实现重命名
> * 移动：当目标是已存在的目录时，将源文件/目录移动到目标目录

**语法与选项**

| 语法     | `mv [选项] 源文件/目录 目标文件/目录` |
| -------- | ------------------------------------- |
| 常用选项 | 说明                                  |
| `-i`     | 目标存在时提示确认（避免覆盖）        |
| `-f`     | 强制覆盖目标（无提示，谨慎使用）      |

**案例**

```bash
[vect@VM-0-11-centos test]$ tree
.
|-- C++
|   `-- hello
|-- he.txt
`-- num.txt

2 directories, 2 files
# 给已存在文件重命名
[vect@VM-0-11-centos test]$ mv he.txt new.txt
[vect@VM-0-11-centos test]$ ls
C++  new.txt  num.txt
[vect@VM-0-11-centos test]$ clear
[vect@VM-0-11-centos test]$ tree
.
|-- C++
|   `-- hello
|-- new.txt
`-- num.txt

2 directories, 2 files
[vect@VM-0-11-centos test]$ mv new.txt hh.txt
[vect@VM-0-11-centos test]$ ls
C++  hh.txt  num.txt
# 给已存在目录改名
[vect@VM-0-11-centos test]$ mv C++ cpp
[vect@VM-0-11-centos test]$ ls
cpp  hh.txt  num.txt
[vect@VM-0-11-centos test]$ pwd
/home/vect/test
# 移动文件到另一个目录
[vect@VM-0-11-centos test]$ mv hh.txt ./cpp
[vect@VM-0-11-centos test]$ ls
cpp  num.txt
[vect@VM-0-11-centos test]$ tree
.
|-- cpp
|   |-- hello
|   `-- hh.txt
`-- num.txt

2 directories, 2 files
[vect@VM-0-11-centos test]$ pwd
/home/vect/test
# 切到上一级目录
[vect@VM-0-11-centos test]$ cd ../
[vect@VM-0-11-centos ~]$ mkdir Linux
[vect@VM-0-11-centos ~]$ tree
.
|-- Linux
|-- list
`-- test
    |-- cpp
    |   |-- hello
    |   `-- hh.txt
    `-- num.txt

5 directories, 2 files
# 移动目录到另一个目录
[vect@VM-0-11-centos ~]$ mv test /home/vect/list
[vect@VM-0-11-centos ~]$ tree
.
|-- Linux
`-- list
    `-- test
        |-- cpp
        |   |-- hello
        |   `-- hh.txt
        `-- num.txt

5 directories, 2 files
[vect@VM-0-11-centos ~]$ 

```

## 2. cp（复制文件 / 目录）

**复制与移动的差异**：`cp`是创建源文件 / 目录的副本（源文件保留），`mv`是转移源文件 / 目录的位置（源文件消失）；复制目录需加`-r`选项（递归复制目录内所有内容）。

**语法与选项**

| 语法     | `cp [选项] 源文件/目录 目标文件/目录`                |
| -------- | ---------------------------------------------------- |
| 常用选项 | 说明                                                 |
| `-r`     | 递归复制目录（含子目录和文件）                       |
| `-i`     | 目标存在时提示确认                                   |
| `-f`     | 强制覆盖目标（无提示）                               |
| `-p`     | 保留源文件的权限、时间戳等属性（默认复制会改变属性） |

**案例**

```bash
[vect@VM-0-11-centos ~]$ tree
.
|-- Linux
`-- list
    `-- test
        |-- cpp
        |   |-- hello
        |   `-- hh.txt
        `-- num.txt

5 directories, 2 files
[vect@VM-0-11-centos ~]$ pwd
/home/vect
[vect@VM-0-11-centos ~]$ cd /home/vect/list/test
# 拷贝文件形成副本
[vect@VM-0-11-centos test]$ cp num.txt num_copy.txt
[vect@VM-0-11-centos test]$ ls
cpp  num_copy.txt  num.txt
# 递归拷贝目录所有文件
[vect@VM-0-11-centos test]$ cp -r cpp cpp_copy
[vect@VM-0-11-centos test]$ ls
cpp  cpp_copy  num_copy.txt  num.txt
# 保留源文件属性拷贝形成副本
[vect@VM-0-11-centos test]$ cp -p num.txt num_cc.txt
[vect@VM-0-11-centos test]$ ls
cpp  cpp_copy  num_cc.txt  num_copy.txt  num.txt
[vect@VM-0-11-centos test]$ pwd
/home/vect/list/test
[vect@VM-0-11-centos test]$ > cnt.txt
[vect@VM-0-11-centos test]$ tree
.
|-- cnt.txt
|-- cpp
|   |-- hello
|   `-- hh.txt
|-- cpp_copy
|   |-- hello
|   `-- hh.txt
|-- num_cc.txt
|-- num_copy.txt
`-- num.txt

4 directories, 6 files
# 把/home/vect/list/test路径下的所有.txt文件拷贝到Linux路径下
[vect@VM-0-11-centos test]$ cp *.txt /home/vect/Linux
[vect@VM-0-11-centos test]$ cd ~
[vect@VM-0-11-centos ~]$ tree
.
|-- Linux
|   |-- cnt.txt
|   |-- num_cc.txt
|   |-- num_copy.txt
|   `-- num.txt
`-- list
    `-- test
        |-- cnt.txt
        |-- cpp
        |   |-- hello
        |   `-- hh.txt
        |-- cpp_copy
        |   |-- hello
        |   `-- hh.txt
        |-- num_cc.txt
        |-- num_copy.txt
        `-- num.txt

7 directories, 10 files
```

## 3. chmod（修改文件 / 目录权限）

### 概念铺垫

* **Linux 权限模型**：
  
  1. **访问者分类**：
     * u（User）：文件拥有者（创建文件的用户）。
     * g（Group）：文件所属组（拥有者所在的用户组）。
     * o（Others）：其他用户（既非拥有者也不在所属组的用户）。
     * a（All）：所有用户（u+g+o）。
  
     **如何确定访问者的身份？**按照拥有者->所属组->其他用户的顺序进行认定，并且只能认定一次：
  
     ![image-20251204111955959](https://gitee.com/binary-whispers/pic/raw/master///20251204193725356.png)
  
  2. **基本权限**：
  
     * r（读权限）：对文件可读取内容；对目录可`ls`查看内容。
     * w（写权限）：对文件可修改内容；对目录可创建 / 删除文件。
     * x（执行权限）：对文件可执行（如脚本）；对目录可`cd`进入。
     * `-`：无对应权限。
  
  3. **权限表示方式**：每三个为一组，表示用户对文件的权限![image-20251204110126645](https://gitee.com/binary-whispers/pic/raw/master///20251204193729845.png)
  
     * 字符表示：如`rw-r--r--`（拥有者rw，所属组 r，其他用户r）。
     * 数值表示：先铺垫一下，权限只有有和无两态，那么就可用`bool`表示![image-20251204111236939](https://gitee.com/binary-whispers/pic/raw/master///20251204193735593.png)

### 语法与两种修改方式

| 修改方式 | 语法                                      | 说明                                                    | 案例                                                         |
| -------- | ----------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| 字符方式 | `chmod [u/g/o/a][+/-/=][r/w/x] 文件/目录` | `+`：添加权限；`-`：移除权限；`=`：设置权限（覆盖原有） | 1. 给所有者加执行权限：`chmod u+x file`2. 移除其他用户的写权限：`chmod o-w file`3. 给所有用户设置 rw 权限：`chmod a=rw flie` |
| 数值方式 | `chmod 数值 文件/目录`                    | 用 3 位八进制数（u、g、o 各 1 位）设置权限              | 1. 设置文件权限为 644（常用）：`chmod 644 file`2. 设置目录权限为 755（常用）：`chmod 755 dir` |

### 关键注意事项

* **目录权限的特殊性**：

  * 目录必须有`x`权限才能`cd`进入（即使有`r`权限也无法进入）。
  * 目录有`w`权限但无`x`权限时，无法`cd`进入，但可通过绝对路径创建 / 删除文件（实际中需避免此配置）。

* **递归修改目录权限**：若需修改目录及所有子目录 / 文件的权限，需加`-R`选项

* **`root`无视权限**

### 案例演示

```bash
[vect@VM-0-11-centos Linux]$ ls
cnt.txt  num_cc.txt  num_copy.txt  num.txt
[vect@VM-0-11-centos Linux]$ ls -l cnt.txt
-rw-rw-r-- 1 vect vect 0 Dec  3 18:01 cnt.txt
# 所有者禁止读写 所属组禁止读写 其他用户禁止读
[vect@VM-0-11-centos Linux]$ chmod u-rw,g-rw,o-r cnt.txt
[vect@VM-0-11-centos Linux]$ ls -l cnt.txt
---------- 1 vect vect 0 Dec  3 18:01 cnt.txt
# 拥有者恢复读写权限 所属组恢复读写权限 其他用户恢复读权限且增加写权限
[vect@VM-0-11-centos Linux]$ chmod u+rw,g+rw,o+rw cnt.txt
[vect@VM-0-11-centos Linux]$ ls -l cnt.txt
-rw-rw-rw- 1 vect vect 0 Dec  3 18:01 cnt.txt
# 所有访问者对文件无任何权限
[vect@VM-0-11-centos Linux]$ chmod a=- cnt.txt
[vect@VM-0-11-centos Linux]$ ls -l cnt.txt
---------- 1 vect vect 0 Dec  3 18:01 cnt.txt
# 所有访问者增加读权限
[vect@VM-0-11-centos Linux]$ chmod a=r cnt.txt
[vect@VM-0-11-centos Linux]$ ls -l cnt.txt
-r--r--r-- 1 vect vect 0 Dec  3 18:01 cnt.txt
[vect@VM-0-11-centos Linux]$ ls -l num.txt
-rw-rw-r-- 1 vect vect 7 Dec  3 18:01 num.txt
# 修改两个文件的所有者和所属组有读写权限 其他用户无权限
[vect@VM-0-11-centos Linux]$ chmod 660 num.txt cnt.txt
[vect@VM-0-11-centos Linux]$ ls -l cnt.txt num.txt
-rw-rw---- 1 vect vect 0 Dec  3 18:01 cnt.txt
-rw-rw---- 1 vect vect 7 Dec  3 18:01 num.txt

```



```bash
[vect@VM-0-11-centos ~]$ ls -ld Linux
drwxrwxr-x 3 vect vect 4096 Dec  4 11:42 Linux
# 把Linux目录下的所有文件递归变为755的权限
[vect@VM-0-11-centos ~]$ chmod -R 755 Linux
[vect@VM-0-11-centos ~]$ ls -ld Linux
drwxr-xr-x 3 vect vect 4096 Dec  4 11:42 Linux
[vect@VM-0-11-centos ~]$ ls -l Linux
total 16
-rwxr-xr-x 1 vect vect    0 Dec  3 18:01 cnt.txt
-rwxr-xr-x 1 vect vect    0 Dec  4 11:42 copy.txt
-rwxr-xr-x 1 vect vect    7 Dec  3 18:01 num_cc.txt
-rwxr-xr-x 1 vect vect    7 Dec  3 18:01 num_copy.txt
-rwxr-xr-x 1 vect vect    7 Dec  3 18:01 num.txt
drwxr-xr-x 2 vect vect 4096 Dec  4 11:41 show
```



## 4. 辅助权限指令：chown/chgrp/umask（所有者、所属组、默认权限）

### 4.1. chown：修改文件 / 目录的所有者

注意：修改文件目录的所有者，先要经过人家的同意，把东西给人家，人家要不要是两回事，而`root`可以无视这些规则，`sudo`就是给指令提权，相当于是`root`级别

* **语法**：`chown [选项] 新所有者 文件/目录`

* **选项**：`-R`：递归修改目录及子内容的所有者。

* **案例**：

  ```bash
  #  Linux目录下的所有文件所有者递归更改为zwr
  [vect@VM-0-11-centos ~]$ sudo chown -R zwr /home/vect/Linux
  [vect@VM-0-11-centos ~]$ ls -l Linux
  total 16
  -rwxr-xr-x 1 zwr vect    0 Dec  3 18:01 cnt.txt
  -rwxr-xr-x 1 zwr vect    0 Dec  4 11:42 copy.txt
  -rwxr-xr-x 1 zwr vect    7 Dec  3 18:01 num_cc.txt
  -rwxr-xr-x 1 zwr vect    7 Dec  3 18:01 num_copy.txt
  -rwxr-xr-x 1 zwr vect    7 Dec  3 18:01 num.txt
  drwxr-xr-x 2 zwr vect 4096 Dec  4 11:41 show
  ```
  

### 4.2. chgrp：修改文件 / 目录的所属组

* **语法**：`chgrp [选项] 新所属组 文件/目录`

* **选项**：`-R`：递归修改。

* **案例**：

  ```bash
  # 将test.txt的所属组改为group1
  [vect@VM-0-11-centos ~]]$ chgrp group1 test.txt
  ```
  

### 4.3. umask：设置新建文件 / 目录的默认权限

* **概念**：Linux 新建文件默认权限为`664`（无执行权限，避免安全风险），新建目录默认权限为`775`（需执行权限以进入）

```bash
[vect@VM-0-11-centos ~]$ mkdir study
[vect@VM-0-11-centos ~]$ ls -ld ./study
# 775
drwxrwxr-x 2 vect vect 4096 Dec  4 15:25 ./study
[vect@VM-0-11-centos ~]$ cd ./study
[vect@VM-0-11-centos study]$ pwd
/home/vect/study
[vect@VM-0-11-centos study]$ touch homework.txt
[vect@VM-0-11-centos study]$ ls -l homework.txt
# 664
-rw-rw-r-- 1 vect vect 0 Dec  4 15:27 homework.txt
```

* **最终权限 = 起始权限 & (~权限掩码)**

| 文件类型 | 起始权限 |
| -------- | -------- |
| 目录     | `777`    |
| 文件     | `666`    |

起始权限把文件权限拉满了，对应的权限掩码默认就是`002`

对于`002`->`000 000 010` 按位取反->`111 111 101`

对于`777`->`111 111 111` ->`111 111 111 & 111 111 101 = 111 111 101`->`775`

对于`666`->`110 110 110`->`110 110 110 & 111 111 101 = 110 110 100`->`664`

所以印证上文新建目录权限默认为`775`,新建文件权限默认为`664`

**这里还有个删除普通文件的问题：**

```bash
[vect@VM-0-11-centos study]$ chmod a=- homework.txt
[vect@VM-0-11-centos study]$ ls -l homework.txt
---------- 1 vect vect 0 Dec  4 15:27 homework.txt
[vect@VM-0-11-centos study]$ rm homework.txt
rm: remove write-protected regular empty file ‘homework.txt’? y
[vect@VM-0-11-centos study]$ ls
[vect@VM-0-11-centos study]$ cd ../
[vect@VM-0-11-centos ~]$ pwd
/home/vect
[vect@VM-0-11-centos ~]$ ls
cpp  Linux  list  study
# 为什么homework.txt没有w权限也能被删除？
[vect@VM-0-11-centos ~]$ ls -l study
total 0
```

文件没有写权限却被删除了->**新建删除修改普通文件的权限，不是文件自己的权限，而是受文件所在目录的权限约束**

```bash
[vect@VM-0-11-centos ~]$ ls -ld study
drwxrwxr-x 2 vect vect 4096 Dec  4 15:40 study
[vect@VM-0-11-centos ~]$ chmod u-w,g-w study
[vect@VM-0-11-centos ~]$ ls -ld study
dr-xr-xr-x 2 vect vect 4096 Dec  4 15:40 study
[vect@VM-0-11-centos ~]$ cd ./study
[vect@VM-0-11-centos study]$ mkdir hh
mkdir: cannot create directory ‘hh’: Permission denied
[vect@VM-0-11-centos study]$ touch hh.c
touch: cannot touch ‘hh.c’: Permission denied

```

* **语法**：

  * 查看 umask：`umask`。
  * 设置 umask：`umask 数值`（如`umask 027`）。

* **案例**：

  ```bash
  # 查看当前umask（root默认022，普通用户002）
  [vect@VM-0-11-centos study]$ umask
  # 设置umask为027（新建文件权限666&~027=640，目录777&~027=750）
  [vect@VM-0-11-centos study]$ umask 027
  ```


## 5. 粘滞位：解决目录 “删除权限” 问题

### 概念铺垫

* **目录权限**

> 可执行权限：无可执行权限，无法`cd`到目录中
>
> 可读权限：无可读权限，无法用`ls`等指令查看目录中的文件
>
> 可写权限：无可写权限，无法在目录中新建和删除文件

* **问题场景**：若目录权限为`777`（所有用户可写），任何用户都能删除目录内的文件（即使不是文件所有者），存在极大的安全风险！！！
* **粘滞位作用**：给目录加粘滞位（`chmod +t 目录`）后，仅 3 类用户可删除目录内文件：
  1. 超级管理员（root）。
  2. 该目录的所有者。
  3. 该文件所有者。

### 语法与案例

```bash
# 1. 给/home目录加粘滞位（系统默认已加，确保普通用户不能删除他人文件）
[root@bite-alicloud ~]$ chmod +t /home
# 2. 验证粘滞位（目录权限最后一位变为t）
[root@bite-alicloud ~]$ ls -ld /home
# 输出：drwxrwxrwt. 3 root root 4096 Jan 11 16:00 /home（t表示粘滞位）
```

## 权限的总结

| 权限        | 核心作用                                                     | 关键约束（易错点）                                           |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `x`（执行） | 目录的「准入通行证」：1. 能否 `cd` 进入目录2. 能否访问 / 操作目录内的文件（即使知道文件名） | ✅ 有 `x`：才能执行目录相关的所有命令（``cd/rm/touch` 等）                                                              ❌ 无 `x`：哪怕有 `r/w`，也无法 `cd` 进入、无法操作目录内任何内容（`ls/rm` 全报错） |
| `r`（读）   | 仅控制「查看目录内文件列表」（ls 列出文件名 / 权限）         | ✅ 有 `r+x`：能 `ls` 列出目录内容                                                                                                     ❌ 有 `r` 无 `x`：无法 `ls`（`x` 是前提）                                                                                                       ❌ 有 `x` 无 `r`：能 `cd` 进入，但 `ls` 提示权限不足（仅能操作**已知名称**的文件，如 `rm /dir/已知文件.txt`） |
| `w`（写）   | 控制「修改目录内文件列表」（增 / 删 / 重命名文件）           | ✅ 有 `w+x`：能` touch `创建、`rm` 删除、`mv`重命名目录内文件                                                               ❌ 有 `w` 无 `x`：无法 `cd` 进入，仅能通过绝对路径创建 / 删除文件（实际无意义，禁止配置）                        ❌ 无 `w`：即使有 `r+x`，也不能增 / 删 / 改目录内文件 |

和普通文件区分开：

| 类型 | `x` 权限含义                  | 核心逻辑                                  |
| ---- | ----------------------------- | ----------------------------------------- |
| 目录 | 准入权（能否进入 / 操作）     | 目录的 `x` 是 “基础”，`r/w` 是 “进阶操作” |
| 文件 | 执行权（能否运行程序 / 脚本） | 文件的 `x` 是 “功能”，`r/w` 是 “读写内容” |



# 五、进阶工具与实用技巧

### 1. 压缩与解压：zip/unzip/tar

###  tar：Linux 最常用压缩工具（支持多种格式）

记住两组即可：

>打包压缩：`tar cvzf fileName.tgz src`
>
>解压到指定目录: `tar xvzf fileName.tgz -C dir`
>
>* `-c`：create 创建压缩文件
>* `-v`：显示压缩/解压过程
>* `-z`：表示正在压缩
>* `-f`：后面是压缩包的名字
>* `-x`：解压压缩包
>* `-C`：指定解压目录

* **常用案例**：

  ```bash
  [vect@VM-0-11-centos ~]$ tree Linux
  Linux
  |-- cnt.txt
  |-- copy.txt
  |-- num_cc.txt
  |-- num_copy.txt
  |-- num.txt
  `-- show
      |-- show1.txt
      `-- show2
  
  1 directory, 7 files
  # 压缩
  [vect@VM-0-11-centos ~]$ tar cvzf Linux.tgz Linux
  Linux/
  Linux/num.txt
  Linux/num_copy.txt
  Linux/cnt.txt
  Linux/num_cc.txt
  Linux/copy.txt
  Linux/show/
  Linux/show/show1.txt
  Linux/show/show2
  [vect@VM-0-11-centos ~]$ pwd
  /home/vect
  # 解压
  [vect@VM-0-11-centos ~]$ tar xvzf Linux.tgz -C ./cpp
  Linux/
  Linux/num.txt
  Linux/num_copy.txt
  Linux/cnt.txt
  Linux/num_cc.txt
  Linux/copy.txt
  Linux/show/
  Linux/show/show1.txt
  Linux/show/show2
  [vect@VM-0-11-centos ~]$ tree cpp
  cpp
  `-- Linux
      |-- cnt.txt
      |-- copy.txt
      |-- num_cc.txt
      |-- num_copy.txt
      |-- num.txt
      `-- show
          |-- show1.txt
          `-- show2
  
  2 directories, 7 files
  
  ```


### zip/unzip

> 打包压缩：`zip -r fileName.zip src`   整个目录下的文件都打包压缩
>
> 解压到指定目录：`unzip -d dir`

### rzsz

这个工具用于Windows和Linux交互：

![image-20251204171539216](https://gitee.com/binary-whispers/pic/raw/master///20251204193745536.png)

![image-20251204171732960](https://gitee.com/binary-whispers/pic/raw/master///20251204193752988.png)

## 2. 命令帮助：man（查看指令手册）

* **概念**：`man`是 Linux 自带的 “联机手册”，可查看任何命令的语法、选项和示例，是从 0 到精通的核心工具。

* **手册章节**：`man`手册分 9 章，常用章节：

  * 1：普通命令（如`man 1 ls`查看`ls`命令）。
  * 2：系统调用（如`man 2 open`查看`open`函数）。
  * 3：库函数（如`man 3 printf`查看`printf`函数）。
  * 8：系统管理命令（如`man 8 ifconfig`查看`ifconfig`）。

* **案例**：

  ```bash
  # 1. 查看ls命令的手册（默认章节1）
  [vect@VM-0-11-centos ~]$ man ls
  # 2. 查看open系统调用的手册（章节2）
  [vect@VM-0-11-centos ~]$ man 2 open
  # 3. 查看man手册的使用说明（查看man自己）
  [vect@VM-0-11-centos ~]$ man man
  ```
  

## 3. 常用热键：提升操作效率

* `Tab`：命令补全 / 文件补全（按一次补全，按两次显示所有可能选项，避免输错命令）。
* `Ctrl+C`：终止当前运行的命令（如终止`tail -f`跟踪日志）。
* `Ctrl+D`：退出当前 Shell 会话（等价于`exit`）。
* `Ctrl+L`：清空终端屏幕（等价于`clear`命令）。
