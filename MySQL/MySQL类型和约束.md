# 数据类型

## 一、整数类型

| 类型        | 字节  | 有符号范围               | 无符号范围       | 常见场景         |
| ----------- | ----- | ------------------------ | ---------------- | ---------------- |
| `tinyint`   | 1字节 | -128 ~ 127               | 0 ~ $2^{8} - 1$  | 状态、枚举小范围 |
| `smallint`  | 2字节 | $-2^{15}$ ~ $2^{15} - 1$ | 0 ~ $2^{16} - 1$ | 小范围数量、等级 |
| `mediumint` | 3字节 | $-2^{23}$ ~ $2^{23} - 1$ | 0 ~ $2^{24}-1$   | 少见             |
| `int`       | 4字节 | -21亿 ~ 21亿             | 0 ~ 42亿         | 常规的id、计数   |
| `bigint`    | 8字节 | 超级大                   | 超级大           | 用户id、订单id   |

**`tinyint`最常见的**

```sql
status tinyint unsigned not null default 0 comment '状态：0正常，1禁用，2删除';
```

适合：

* 用户状态
* 订单状态
* 性别
* 是否删除
* 小范围枚举

**`bigint`适合用户量极大，点击量极大的场景，数据量增长大，`int`很容易不够用，一次性开够**

适合：

* 用户id
* 订单id
* 支付流水id
* 消息id

```sql
user_id bigint unsigned not null comment '用户id'
order_id bigint unsigned not null comment '订单id'
```

## 二、浮点类型和定点类型

值得注意的是：**无符号类型的浮点和定点类型，直接砍掉负数部分，范围是0 ~ XX.XX**

### `float(m,d)`

* `m`：总位数
* `d`：小数位数

比如说：`float(5,2)`，总共五位数，小数点占两位，最大到`999.99`

需要注意：`float`是浮点数，存在精度误差

###  `double`

`double`比`float`精度高，占8字节，但是依旧是浮点数，存在精度误差

适合：

* 科学计算
* 统计估算
* 经纬度

###  `decimal`

`decimal`可以规避精度损失

涉及到金融、电商、支付、金钱的业务，务必用`decimal`，要保证一分都不能出错

```sql
amout decimal(10, 2) not null default 0.00 comment '金额'
```

* 总共10位
* 小数精确到2位
* 整数最多8位
* 最大值约为`99999999.99`

常见的有：

```sql
price decimal(10,2) not null default 0.00 comment '商品价格'
balance decimal(18,2) not null default 0.00 comment '账户余额'
rate decimal(5,4) not null default 0.0000 comment '费率'
```

## 三、字符串类型

###  `char(n)`

**固定长度字符串，给多少就开多少空间**

例如：

```sql
gender char(1) not null comment '性别：m男，f女'
```

适合：

* 固定长度的编码
* 性别
* 手机区号

###  `varchar(n)`

**边长字符串，括号里的是字符数上限，实际用多少占用多少空间**

`n`是字符数，而不是字节数，底层会根据当前的字符集算出实际占用多少字节：

* `utf8`占用3字节
* `utf8mb4`占用4字节，支持emoji

`varchar`可以指定0~65535之间的值，但是有1 - 3字节用于记录数据大小，所以有效字节数是65532

编码是`utf8`时，`n`的最大值是$65532 \div 3 = 21844$，所以最多只能存21844个字符

适合：

* 用户名
* 昵称
* 邮箱
* 标题
* 地址
* URL

例如：

```sql
username varchar(64) not null comment '用户名'
email varchar(128) default null comment '邮箱'
nickname varchar(64) not null default '' comment '昵称'
title varchar(255) not null comment '标题'
```

###  `text`大文本

见类型：

| 类型         | 最大长度 | 场景                 |
| :----------- | -------: | :------------------- |
| `tinytext`   | 255 字节 | 很短文本             |
| `text`       |     64KB | 文章内容、描述       |
| `mediumtext` |     16MB | 长文章               |
| `longtext`   |      4GB | 超长文本，很少直接用 |

例如：

```sql
content text comment '文章内容'
description text comment '商品描述'
```

需要注意：**大字段尽量不要和高频查询字段放在同一张表中**

* 大字段会影响缓存命中
* 这样可以减少IO，提高效率

##  四、日期时间类型

###  `date`

```sql
birthday data default null comment '生日'
```

适合：

* 生日
* 日期
* 统计日期
* 入职日期

格式：`YYYY-MM-DD`，占用3字节

###  `time`

```sql
duration time comment '时长'
```

格式：`HH:MM:SS`

适合：

* 时间间隔
* 当天时间点
* 持续时间

存**耗时**使用整数秒/毫秒：

```sql
duration_ms int unsigned not null default 0 comment '耗时毫秒'
```

###  `datetime`

占用8字节，适合：

* 创建时间
* 更新时间
* 业务发生时间
* 订单时间

范围很大：`1000-01-01 ~ 9999-12-31`

不受时区转换影响，这个底层逻辑是，把这个时间**刻在硬盘上**，硬盘带到哪个时区，读出来的时间永远不变

###  `timestamp`

本质是一个时间戳，不存年月日，存的是一个**整数**，从 1970 年 1 月 1 日 0 点 0 分 0 秒（UTC 零时区）到现在，**一共过了多少秒**

会随主机的时区变化：

* 存入：MySQL会先看当前系统处于什么时区，把传进来的时间**换算成时间戳**，存入硬盘
* 读取：MySQL把硬盘那串时间戳秒数逃出来，再根据查询的客户端所在时区，**换算**成当地时间展示

**`timestamp`只能存到2038年**

`timestamp`底层是一个4字节的带符号整数，最大值$2^{31} - 1 = 2147483647$，时间到了**2038年1月19日凌晨 03:14:07**时，从1970年开始算的秒数刚好是$2147483647$，再往后走一秒，溢出了



##  五、`enum`和`set`

### `enum`枚举，多选一

```sql
gender enum('male', 'female', 'unknown') not null default 'unknown'
```

内部是索引：

```text
male -> 1
female -> 2
unknown -> 3
```

可以这样查询：

```sql
select * from user where gender = 'male'
```

查看某个枚举值：

```sql
select * from user where gender = 'male'
```

查询某个字段是否等于几个枚举值：

```sql
select * from user where gender in ('male', 'unknown')
```

**`enum`存在以下问题：**

* 新增枚举值就需要改变表结构
* 跨语言、跨服务维护成本高

### `set`不定项选择，类似位图

```sql
hobby set('music', 'sport', 'game', 'read')
```

底层类似位图，查询可以：

```sql
select * from user where find_in_set('music', hobby)
# 或者是
select * from user where hobby & 1
```

# 约束

##  非空约束

`not null` 约束用户必须插入一个值

```sql
username varchar(64) not null commnet '用户名'
status tinyint unsigned not null default 0 comment '状态'
created_at datetime not null default current_timestamp commnet '创建时间'
```

`null`不参与运算，代表没有，而不是空值，空值是 **` `**

## 缺省值

`default`的使用：

* 当用户忽略这一列时，用这个缺省值
* 没有缺省值且约束了`not null`，忽略这一列会报错

使用中，建议：

数字：

```sql
conut int unsigned not null default 0
```

字符串：

```sql
nickname varchar(64) not null default ''
```

状态：

```sql
status tinyint unsigned not null default 0
```

时间：

```sql
created_at datetime not null default current_timestamp
```

## 注释

`comment`是写给程序员的，协作开发一定要写注释

## 主键约束

一张表最多一个主键，但是主键可以由多列组成，变成复合主键

### 单列主键

```sql
id bigint unsigned not null primary key comment '主键id'
```

要求：

* 唯一
* 非空
* 不要频繁修改

主键为什么通常使用整数？

* 整数比字符串省空间
* 整数查找比字符串块
* 自增主键插入顺序友好

### 自增主键

```sql
id bigint unsigned not null auto_increment primary key
```



`auto_increment`对应的字段不给值，会自动被系统触发，系统会从当前字段中已经有的最大值+1，得到一个新的不同的值，搭配主键使用

特点：

* 自增长字段是整数
* 一张表最多只能有一个自增长
* 任何字段做自增长，必须是一个索引

### 复合主键

多种列属性是强绑定的，比如课程名称和课程号是一一对应的，那么就可以设置成复合主键

以下的案例是，一个学生不能重复选一门课程

```sql
create table student_course(
	student_id bigint unsigned not null,
    course_id bigint unsigned not null,
    score int default null,
    primary key (student_id, course_id)
)
```

## 唯一键

**主键标识行唯一，唯一键保证业务字段不重复**

比如说有个手机号，身份证号，这些都得保证每个人的属性都是不同的，不能重复

```sql
phone varchar(20) not null unique comment '手机号'
```

唯一键可以有多个`null`，因为`null`不参与运算，不计数

## 外键

**保证两张表之间的数据关系不能乱**

举个例子：用户表和订单表

一个用户可以有多个订单，所以用户表是父表，订单表是子表

用户`users`：

```sql
create table users (
	id bigint primary key,
    name varchar(64)
);
```

订单表`orders`:

```sql
create table orders(
	id bigint primary key,
    user_id bigint,
    amount decimal(10,2),
    foreign key (user_id) references users(id)
);
```

`foreign key (user_id) references users(id)`这一行规定了，`order.user_id`的值必须在`users.id`里面存在

插入两个真实用户：

```sql
mysql> insert into users values(1, 'kunkun');
Query OK, 1 row affected (0.10 sec)

mysql> insert into users values(2, 'lele');
Query OK, 1 row affected (0.01 sec)
```

插入合法订单：

```sql
mysql> insert into orders values (101, 1, 99.50);
Query OK, 1 row affected (0.03 sec)

mysql> insert into orders values (102, 2, 110);
Query OK, 1 row affected (0.02 sec)
```

**插入不存在的用户，会报错：**

```sql
mysql> insert into orders values (12, 3, 120);
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails 
```

子表`orders`想插入`user_id = 3`，但是父表中`users`不存在这个id，拦截

**外键不允许子表引用不存在的父表数据**

**修改订单归属，会报错：**

```sql
mysql> update orders set user_id = 888 where id = 101;
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails
```

**外键不允许子表把引用改成不存在的父表数据**

**删除被引用的用户，会报错**

```sql
mysql> delete from users where id = 1;
ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails
```

**外键不允许父表删除仍被子表引用的数据**->得先把数据删干净

总结一下外键的作用：

子表里引用父表的值，必须真实存在，不能乱改，不能断关系
