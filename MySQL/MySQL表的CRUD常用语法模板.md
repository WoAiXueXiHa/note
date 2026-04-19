CRUD 的核心原则非常简单：**保护数据（不乱删、不误改）、节约资源（不乱查、不循环插）**

## 0. 基础业务表定义

定义一张极其简单的 user_basic 表，没有任何多余的索引，只有主键和几个日常极常用的字段类型

**操作模板：**

```sql
create table 表名 (
  字段名 数据类型 约束 注释,
  ...
  primary key (主键)
) engine=存储引擎 default charset=字符集;
```

**实战操作：**

```sql
create table user_basic (
  id bigint unsigned not null auto_increment comment '主键',
  username varchar(64) not null comment '用户名',
  age int not null default 0 comment '年龄',
  status tinyint not null default 1 comment '状态: 0-禁用, 1-正常',
  primary key (id)
) engine=innodb default charset=utf8mb4 comment='用户基础表';
```

------

## 1. [C] 增 (create)

增加数据的核心场景就两个：要么一条一条插，要么一次性插一堆

### 1.1 新增一条数据

* **使用场景：** 用户填完表单点击注册，后台生成一条记录

* **问题：** 为什么规范要求必须显式写出列名，而不是直接 `insert into table values(...)`？（防止未来表结构增加字段时，代码里的 SQL 直接报错）

* **语法模板：** `insert into 表名 (列名1, 列名2) values (值1, 值2);`

* **实战操作：**

  ```sql
  insert into user_basic (username, age) values ('zhangsan', 22);
  ```

### 1.2 新增一批数据 (批量插入)

* **使用场景：** 运营在后台上传了一个 Excel 表格，要求把里面的 100 个用户导入系统 

* **问题：** 代码里用 for 循环执行 100 次单条 insert，和用一条批量 insert 语句有什么区别？（批量插入只需和数据库建立一次网络交互，极大地节省了网络耗时） 

* **语法模板：** `insert into 表名 (列名) values (值组1), (值组2), ...;`

* **实战操作：**

  ```sql
  insert into user_basic (username, age) 
  values 
    ('lisi', 24), 
    ('wangwu', 25),
    ('zhaoliu', 18);
  ```

------

## 2. [R] 查 (read)

查询是写得最多的 SQL 日常开发的规矩是：要什么查什么，要多少取多少

### 2.1 查询指定字段与主键查询

* **使用场景：** 只需要展示用户的名字和年龄，不需要看状态

* **tips：** 不建议用 `select *`->浪费数据库内存，增加网络传输带宽压力

* **语法模板：** `select 字段列表 from 表名 where 过滤条件;`

* **实战操作：**

  ```sql
  select username, age from user_basic where id = 1;
  ```

### 2.2 多条件组合查询

* **使用场景：** 筛选出“年龄大于20岁”且“账号状态正常”的用户 

* **问题：** `and` 和 `or` 混用时的优先级 （`and` 优先级高于 `or`，强烈建议用括号 `()` 明确优先级，避免歧义）

* **语法模板：** `select 字段 from 表名 where 条件1 and/or 条件2;`

* **实战操作：**

  ```sql
  select id, username from user_basic where age > 20 and status = 1;
  ```

### 2.3 模糊查询

* **使用场景：** 搜索框里输入 "zhang"，想搜出 "zhangsan" 或者 "zhangxueyou" 

* **语法模板：** `select 字段 from 表名 where 字段 like '匹配串';` (`%` 代表任意字符)

* **实战操作：**

  ```sql
  select id, username from user_basic where username like 'zhang%';
  ```

### 2.4 排序与分页查询

* **使用场景：** 后台列表页，按年龄从大到小排，每页只展示 10 个人 

* **速记：** 分页公式是什么（`limit (页码 - 1) * 每页数量, 每页数量`）

* **语法模板：** `select 字段 from 表名 order by 排序列 [asc|desc] limit 偏移量, 限制数;`

* **实战操作：**

  ```sql
  select id, username, age 
  from user_basic 
  where status = 1 
  order by age desc 
  limit 0, 10;
  ```

### 2.5 统计查询

* **使用场景：** 在数据看板上显示“当前一共有多少个正常用户” 

* **问题：** `count(1)` 和 `count(列名)` 有什么区别？（`count(列名)` 如果那一行该列的值是 null，就不会被统计进去） 

* **语法模板：** `select count(1) from 表名 where 条件;`

* **实战操作：**

  ```SQL
  select count(1) as total_active_users from user_basic where status = 1;
  ```

------

## 3. [U] 改 (update)

修改操作的绝对底线：永远不要忘记写 `where` 条件

### 3.1 更新单个或多个字段

* **使用场景：** 用户在资料页修改了自己的名字和年龄

* **语法模板：** `update 表名 set 字段1 = 新值, 字段2 = 新值 where 精确条件;`

* **实战操作：** 把 id 为 1 的用户，名字改成 "zhangsan_v2"，年龄改为 23

  ```sql
  update user_basic 
  set username = 'zhangsan_v2', age = 23 
  where id = 1;
  ```

### 3.2 批量条件更新

* **使用场景：** 业务逻辑调整，需要把所有未成年（小于18岁）的用户账号强制改为禁用状态（状态置为 0）

* **语法模板：** `update 表名 set 字段 = 新值 where 范围条件;`

* **实战操作：**

  ```sql
  update user_basic set status = 0 where age < 18;
  ```

------

## 4. [D] 删 (delete)

实际业务中，除非是测试环境清数据，或者清理毫无价值的系统日志，否则极少使用纯粹的物理删除

### 4.1 物理删除 (真删)

* **使用场景：** 真的写错数据了，或者测试完清理现场 

* **问题：** `delete` 语句忘记写 `where` 会怎样？（全表数据被清空，造成严重生产事故） 

* **语法模板：** `delete from 表名 where 条件;`

* **实战操作：**

  ```sql
  delete from user_basic where id = 99;
  ```

### 4.2 逻辑删除 (假删，最常用)

* **使用场景：** 用户点击了“注销账号”，实际上只是把状态打了个标记，数据还留在硬盘上以便后续追溯

* **语法模板：** 本质上是 update 语句

* **实战操作：** 把 id 为 2 的用户“删除”

  ```sql
  update user_basic set status = 0 where id = 2;
  ```

------

## 总结：一条数据的 CRUD 极简一生

在最基础的业务代码里，一个用户的生命周期闭环如下：

1. **[C]** 来新用户了： `insert into user_basic (username, age) values ('jack', 20);`
2. **[R]** 查一下他在不在： `select id, username from user_basic where username = 'jack';`
3. **[U]** 他长大了： `update user_basic set age = 21 where username = 'jack';`
4. **[D]** 他退出了： `update user_basic set status = 0 where username = 'jack';` (逻辑删除) 或 `delete from user_basic where username = 'jack';` (物理删除)
