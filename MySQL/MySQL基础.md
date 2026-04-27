# MySQL基础

## 什么是MySQL？

![image-20260426211138814](https://gitee.com/binary-whispers/pic/raw/master///20260426211140987.png)

* mysqld是数据库的服务端

* mysql是数据库的客户端

* mysql是一种基于CS模式，提供了一套数据存储服务的网络服务

  ![image-20260426211546297](https://gitee.com/binary-whispers/pic/raw/master///20260426211549224.png)

* 而数据库是在磁盘或内存中存储的特定数据结构的方法

MySQL是世界上最受欢迎的免费开源的项目，上手比较简单

## 为啥要数据库？

文件保存数据存在缺点：

* 安全性
* 不利于数据查询和管理--比如我想查询某个字段，很困难
* 不利于存储海量数据

数据库是为了高效解决数据存储和查询的设计

## 服务器、数据库、表关系三者

![image-20260426212158353](https://gitee.com/binary-whispers/pic/raw/master///20260426212200568.png)

## MySQL的架构

详细架构图如下：

![image-20260426213348114](https://gitee.com/binary-whispers/pic/raw/master///20260426213349716.png)

注：上图源自网络

初学阶段先理解三层架构：

![image-20260426213625374](https://gitee.com/binary-whispers/pic/raw/master///20260426213627852.png)

MySQL是一种网络服务，那么就可以把它当成一家餐厅：

* **第一层：连接层（前台）**

  * 监听新连接，账号密码校验，管理连接

* **第二层：服务层（后台）**

  * 负责所有**逻辑处理**，不碰真实的数据：
    * 语法检查：保证每条命令的改动都是安全且正确
    * 查询优化：帮助用户选择最高效的方式
    * 权限判断：判断当前操作你是否有权限，无权限直接拦截

  服务层主要负责SQL解析，优化

* **第三层：存储引擎**

  * 真正**存数据、读数据**的地方
    * InnoDB引擎：支持事务、锁、崩溃恢复
    * MyISAM引擎：不支持事务

  服务层发指令，引擎层去硬盘/内存存取数据

## SQL语句分类

DDL：data definition language 数据定义，维护存储数据的结构

DML：data manipulation language 数据操作，对表中数据操作

DCL：data control language 数据控制，权限和事务的管理

# 库的基本操作

## C 新增/创建一个数据库

```sql
create database db_name [各种选项]
```

![image-20260426214326527](https://gitee.com/binary-whispers/pic/raw/master///20260426214328243.png)

在Linux中创建一个目录

这里选项着重介绍一下数据库的两个编码集：

* 数据库编码集：数据库未来存储数据的编码格式
* 数据库校验集：数据库读取时采用的编码格式

**要保证操作时用的同一个编码格式**

```sql
# 查看当前的默认字符集和校验集
show variables like 'character_set_database';
show variables like 'collation_database';
```

## R 查看数据库

```sql
show databases;		# 显示所有数据库
show create database db_name;	# 显示创建数据库时的语句
use db_name;	# 使用这个数据库
select database();	# 查看当前在哪个数据库
```

## U 修改数据库

```sql
alter database db_name [选项]
# 例如修改校验集和字符集
alter database test_db charset=gbk collate=gbk_chinese_ci;
```

## D 删除数据库

```sql
drop database db_name;
```

删除之后整个目录里的所有文件全没了，**不要随意删库**

![image-20260426215425788](https://gitee.com/binary-whispers/pic/raw/master///20260426215427350.png)

## 备份和恢复数据库

备份：

```sql
mysqldump -P3306 -u root -p 密码 -B 数据库名1 数据库名2... > 新的路径
```

直接把所有有效操作的命令备份了！

只备份一个库里的几个表：

```sql
mysqldump -P3306 -u root -p 密码 -B 数据库名 表一 表二... > 新的路径
```

还原：

```sql
source 备份的文件.sql
```

如果在备份时，没有`-B`的选项，恢复数据库时，要先创建空的数据库，然后使用这个数据库，再使用`source`还原

# 表操作

## C 创建表

```sql
create table table_name(
	列名 列属性,
    列名 列属性,
	列名 列属性
) [character set 字符集 collate 校验集 engine 存储引擎];
```

## R 查看表

```sql
show tables;	#查看所有表

desc table_name;	#查看详细信息

show create table table_name\G;	#查看创建时的详细信息
```

## U 修改表

```sql
# 新增一列
alter table table_name add 列名 列属性;

# 修改一列的属性
alter table table_name modify 列名 新数据类型;

# 删除这一列
alter table table_name drop 列名;

# 修改表名称
alter table table_name rename to  新的表名;

# 修改列名称
alter table_name change 旧列明 新列名 属性;

# 指定字段插入
insert into 表名字(字段1,字段2...)  values(值1,值2...);

# 全列插入，没有省略字段
insert into 表名字 values(值1,值2...);
```

## 删除表

```sql
drop table table_name;
```



