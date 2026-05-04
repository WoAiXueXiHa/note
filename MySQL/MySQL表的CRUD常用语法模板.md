# 增

## 本质

MySQL 的“增”主要是 `insert`，本质是：

```text
向表中新增一行或多行记录。
```

先准备一张练习表：

```sql
create table student (
    id int primary key auto_increment,
    name varchar(20) not null,
    age int,
    gender varchar(10),
    score int,
    class_id int,
    phone varchar(20) unique,
    created_at datetime default current_timestamp
);
```

字段含义：

| 字段 | 含义 |
|---|---|
| id | 主键，自增 |
| name | 姓名，不能为空 |
| age | 年龄 |
| gender | 性别 |
| score | 成绩 |
| class_id | 班级 |
| phone | 手机号，唯一 |
| created_at | 创建时间，默认当前时间 |

## 全列插入

```sql
insert into student values(
    1,
    '张三',
    18,
    '男',
    90,
    1,
    '13800138000',
    '2026-05-04 10:00:00'
);
```

不推荐全列插入，因为必须严格依赖表字段顺序，表结构一变就容易出错。

## 指定列插入

推荐写法：

```sql
insert into student (name, age, gender, score, class_id, phone)
values ('张三', 18, '男', 90, 1, '13800138000');
```

这里没有写 `id` 和 `created_at`，因为：

```text
id 会自增生成。
created_at 有默认值，会自动填当前时间。
```

核心原则：

```text
插入时明确列名，不要依赖表结构顺序。
```

## 多行插入

```sql
insert into student (name, age, gender, score, class_id, phone)
values
('李四', 19, '男', 85, 1, '13800138001'),
('王五', 18, '女', 92, 1, '13800138002'),
('赵六', 20, '男', 76, 2, '13800138003'),
('孙七', 19, '女', 88, 2, '13800138004'),
('周八', 21, '男', 60, 3, '13800138005'),
('吴九', 18, '女', null, 3, '13800138006');
```

多行插入比多次单行插入更高效。

## 主键和唯一键冲突

`id` 是主键，`phone` 是唯一键，它们都不能重复。

如果已有：

| id | name | phone |
|---|---|---|
| 1 | 张三 | 13800138000 |

再插入相同 `id`：

```sql
insert into student (id, name, age, gender, score, class_id, phone)
values (1, '小明', 18, '男', 80, 1, '13800138999');
```

会主键冲突。

再插入相同 `phone`：

```sql
insert into student (name, age, gender, score, class_id, phone)
values ('小明', 18, '男', 80, 1, '13800138000');
```

会唯一键冲突。

## `on duplicate key update`

本质：

```text
没冲突就插入，有主键或唯一键冲突就更新原记录。
insert into student (id, name, age, gender, score, class_id, phone)
values (1, '张三', 19, '男', 95, 1, '13800138000')
on duplicate key update
age = 19,
score = 95;
```

如果 `id = 1` 已经存在，就不会插入新行，而是把原记录的 `age` 改成 19，`score` 改成 95。

常见业务写法：手机号存在则更新，不存在则插入。

```sql
insert into student (name, age, gender, score, class_id, phone)
values ('陈十一', 20, '女', 99, 2, '13800138008')
on duplicate key update
age = 20,
score = 99,
class_id = 2;
```

## `replace into`

本质：

```text
没冲突就插入，有冲突就先删除旧记录，再插入新记录。
replace into student (id, name, age, gender, score, class_id, phone)
values (1, '张三新版', 20, '男', 100, 2, '13800138000');
```

注意：`replace into` 不是普通更新，而是 `delete + insert`，可能影响自增值、外键、触发器和未指定字段。

对比：

| 操作 | 没冲突 | 有冲突 |
|---|---|---|
| insert | 插入 | 报错 |
| insert ... on duplicate key update | 插入 | 更新原记录 |
| replace into | 插入 | 删除旧记录再插入 |

## 增的练习

插入一名学生：

```sql
insert into student (name, age, gender, score, class_id, phone)
values ('陈十一', 19, '女', 91, 2, '13800138008');
```

一次插入三名学生：

```sql
insert into student (name, age, gender, score, class_id, phone)
values
('刘一', 18, '男', 80, 1, '13800138009'),
('钱二', 19, '女', 87, 2, '13800138010'),
('冯三', 20, '男', 73, 3, '13800138011');
```

手机号存在则更新，不存在则插入：

```sql
insert into student (name, age, gender, score, class_id, phone)
values ('陈十一', 20, '女', 99, 2, '13800138008')
on duplicate key update
age = 20,
score = 99,
class_id = 2;
```

## 增的总结

```text
insert = 新增行。
指定列插入比全列插入安全。
on duplicate key update = 有则更新，无则插入。
replace into = 删除旧行，再插入新行。
```

# 查

## 本质

MySQL 的“查”主要是 `select`，本质是：

```text
从表中取出满足条件的数据，并按照指定方式展示。
```

常见结构：

```sql
select 列
from 表
where 条件
order by 排序字段
limit 数量;
```

可以理解为：

```text
from 找表
where 筛行
select 选列
distinct 去重
order by 排序
limit 限制数量
```

## 查询所有列

```sql
select * from student;
```

不推荐滥用 `select *`，原因：

```text
1. 读取不需要的字段。
2. 浪费 IO 和网络资源。
3. 表结构变化会影响结果。
4. 不利于性能优化。
```

更推荐：

```sql
select id, name, score
from student;
```

## 查询指定列

```sql
select name, age, score
from student;
```

本质：

```text
只取自己需要的字段。
```

## 别名

```sql
select
    name as 姓名,
    age as 年龄,
    score as 成绩
from student;
```

`as` 可以省略：

```sql
select
    name 姓名,
    age 年龄,
    score 成绩
from student;
```

别名只影响查询结果展示，不修改表结构。

## select 支持表达式

```sql
select
    name,
    score,
    score + 10 as new_score
from student;
```

注意：这只是展示加 10 后的结果，不会真正修改表中数据。

## distinct 去重

```sql
select distinct class_id
from student;
```

多列去重：

```sql
select distinct class_id, gender
from student;
```

它去重的是 `(class_id, gender)` 这个组合。

## where 筛选行

```sql
select id, name, score
from student
where score >= 90;
```

`where` 的本质：

```text
从所有行中筛选满足条件的行。
```

## 比较运算符

等于：

```sql
select * from student where name = '张三';
```

不等于：

```sql
select * from student where gender != '男';
select * from student where gender <> '男';
```

范围比较：

```sql
select * from student where score > 80;
select * from student where age <= 18;
```

## NULL

`NULL` 表示未知、没有值，不等于 0，也不等于空字符串。

错误写法：

```sql
select *
from student
where score = null;
```

正确写法：

```sql
select *
from student
where score is null;
```

不为空：

```sql
select *
from student
where score is not null;
```

MySQL 的 NULL 安全比较：

```sql
select NULL <=> NULL;
```

结果是 1。

实际开发中，更推荐用 `is null` 和 `is not null`。

## between and

```sql
select *
from student
where score between 80 and 90;
```

等价于：

```sql
select *
from student
where score >= 80 and score <= 90;
```

注意：`between A and B` 是闭区间 `[A, B]`。

## in

```sql
select *
from student
where class_id in (1, 2);
```

等价于：

```sql
select *
from student
where class_id = 1 or class_id = 2;
```

反向：

```sql
select *
from student
where class_id not in (1, 2);
```

注意：`not in` 遇到 `NULL` 时要小心。

## like 模糊匹配

| 符号 | 含义 |
|---|---|
| `%` | 任意长度字符，可以是 0 个或多个 |
| `_` | 一个任意字符 |

姓孙：

```sql
select * from student where name like '孙%';
```

包含“小”：

```sql
select * from student where name like '%小%';
```

以“三”结尾：

```sql
select * from student where name like '%三';
```

姓孙且名字两个字：

```sql
select * from student where name like '孙_';
```

## 逻辑运算符

`and`：两个条件都成立。

```sql
select *
from student
where score >= 80 and gender = '女';
```

`or`：满足一个条件即可。

```sql
select *
from student
where score >= 90 or class_id = 2;
```

`not`：条件取反。

```sql
select *
from student
where not gender = '男';
```

`and` 和 `or` 混用时，建议加括号：

```sql
select *
from student
where (gender = '女' or class_id = 1)
  and score >= 90;
```

## order by 排序

升序：

```sql
select name, score
from student
order by score asc;
```

`asc` 默认可以省略：

```sql
select name, score
from student
order by score;
```

降序：

```sql
select name, score
from student
order by score desc;
```

多字段排序：

```sql
select name, class_id, score
from student
order by class_id asc, score desc;
```

含义：先按班级升序，同班级内按成绩降序。

`NULL` 参与排序时通常比任何非 NULL 值都小。

## limit 分页

查询前三条：

```sql
select *
from student
limit 3;
```

从下标 0 开始取 3 条：

```sql
select *
from student
limit 3 offset 0;
```

等价于：

```sql
select *
from student
limit 0, 3;
```

分页公式：

```text
offset = (pageNo - 1) * pageSize
```

第一页，每页 3 条：

```sql
select *
from student
order by id asc
limit 3 offset 0;
```

第二页，每页 3 条：

```sql
select *
from student
order by id asc
limit 3 offset 3;
```

第 5 页，每页 10 条：

```sql
select *
from student
order by id asc
limit 10 offset 40;
```

## 查的练习

查询所有学生姓名和成绩：

```sql
select name, score
from student;
```

查询所有女生：

```sql
select id, name, gender
from student
where gender = '女';
```

查询成绩大于等于 90 的学生：

```sql
select id, name, score
from student
where score >= 90;
```

查询 1 班成绩大于 80 的学生：

```sql
select id, name, score, class_id
from student
where class_id = 1 and score > 80;
```

查询成绩在 80 到 90 之间的学生：

```sql
select id, name, score
from student
where score between 80 and 90;
```

查询 1 班和 2 班学生：

```sql
select id, name, class_id
from student
where class_id in (1, 2);
```

查询成绩前三名：

```sql
select id, name, score
from student
where score is not null
order by score desc
limit 3;
```

## 查的总结

```text
select = 选择列。
from = 指定表。
where = 筛选行。
distinct = 去重。
order by = 排序。
limit = 限制返回数量。
```

重点：

```text
不要滥用 select *。
NULL 用 is null 判断。
and 和 or 混用要加括号。
between 是闭区间。
limit 的 offset 从 0 开始。
```

# 改

## 本质

MySQL 的“改”主要是 `update`，本质是：

```text
先找到满足条件的行，再修改这些行中的某些列。
```

语法：

```sql
update 表名
set 列名 = 新值
where 条件;
```

必须分两步理解：

```text
where 定位要修改哪些行。
set 指定这些行的哪些列改成什么值。
```

## 修改单个字段

把张三成绩改成 95：

```sql
update student
set score = 95
where name = '张三';
```

如果表中有多个张三，这条 SQL 会修改多个张三。

更推荐用主键或唯一键定位：

```sql
update student
set score = 95
where id = 1;
```

或者：

```sql
update student
set score = 95
where phone = '13800138000';
```

## 修改多个字段

把张三年龄改成 19，成绩改成 96：

```sql
update student
set age = 19,
    score = 96
where name = '张三';
```

## 使用表达式修改

给 1 班所有学生加 5 分：

```sql
update student
set score = score + 5
where class_id = 1;
```

本质：

```text
在原来的 score 基础上加 5。
```

## 修改为 NULL

把吴九成绩设置为空：

```sql
update student
set score = null
where name = '吴九';
```

注意：`NULL` 不是 0。

如果写：

```sql
update student
set score = 0
where name = '吴九';
```

表示吴九成绩就是 0 分。

## update 一定要小心 where

危险写法：

```sql
update student
set score = 0;
```

这会把整张表所有学生成绩都改成 0。

正确习惯：

```text
先 select 确认范围。
再 update 修改数据。
最后 select 验证结果。
```

例如给 2 班学生加 5 分：

```sql
select id, name, score
from student
where class_id = 2;
```

确认后再改：

```sql
update student
set score = score + 5
where class_id = 2;
```

改完再查：

```sql
select id, name, score
from student
where class_id = 2;
```

## 用主键或唯一键修改更安全

不推荐：

```sql
update student
set score = 100
where name = '张三';
```

因为 `name` 不一定唯一。

推荐：

```sql
update student
set score = 100
where id = 1;
```

或者：

```sql
update student
set score = 100
where phone = '13800138000';
```

## update 配合多个条件

把 1 班中成绩低于 80 的学生统一改为 80：

```sql
update student
set score = 80
where class_id = 1 and score < 80;
```

## update 配合 in

把 1 班和 2 班学生统一加 3 分：

```sql
update student
set score = score + 3
where class_id in (1, 2);
```

## update 配合 between

把成绩在 60 到 70 之间的学生统一加 10 分：

```sql
update student
set score = score + 10
where score between 60 and 70;
```

注意：`between 60 and 70` 包含 60 和 70。

## update 配合 like

把所有姓孙的学生班级改成 3 班：

```sql
update student
set class_id = 3
where name like '孙%';
```

## update 配合 is null

把成绩为空的学生成绩改成 0：

```sql
update student
set score = 0
where score is null;
```

## update 配合 order by 和 limit

只把一个成绩为空的学生改成 60：

```sql
update student
set score = 60
where score is null
limit 1;
```

按照 id 升序，只修改第一个成绩为空的学生：

```sql
update student
set score = 60
where score is null
order by id asc
limit 1;
```

实际开发中是否允许 `update limit`，要看团队规范。

## 改的练习

把张三成绩改成 95：

```sql
select id, name, score
from student
where name = '张三';

update student
set score = 95
where name = '张三';

select id, name, score
from student
where name = '张三';
```

把手机号为 `13800138008` 的学生成绩改成 98：

```sql
select id, name, score, phone
from student
where phone = '13800138008';

update student
set score = 98
where phone = '13800138008';

select id, name, score, phone
from student
where phone = '13800138008';
```

给 2 班所有学生加 5 分：

```sql
select id, name, score
from student
where class_id = 2;

update student
set score = score + 5
where class_id = 2;

select id, name, score
from student
where class_id = 2;
```

把所有成绩为空的学生改成 60：

```sql
select id, name, score
from student
where score is null;

update student
set score = 60
where score is null;

select id, name, score
from student
where score = 60;
```

把 3 班成绩低于 70 的学生加 10 分：

```sql
select id, name, score, class_id
from student
where class_id = 3 and score < 70;

update student
set score = score + 10
where class_id = 3 and score < 70;

select id, name, score, class_id
from student
where class_id = 3;
```

## 改的总结

修改模板：

```sql
update 表名
set 列名 = 新值
where 条件;
```

修改多个字段：

```sql
update 表名
set 列1 = 新值1,
    列2 = 新值2
where 条件;
```

基于旧值修改：

```sql
update 表名
set 列名 = 列名 + 数值
where 条件;
```

核心：

```text
update = where 定位行 + set 修改列。
```

安全习惯：

```text
1. update 前先 select。
2. update 一定要注意 where。
3. 尽量用主键或唯一键定位。
4. 修改后再 select 验证。
```

危险写法：

```sql
update student
set score = 0;
```

安全写法：

```sql
update student
set score = 0
where id = 1;
```

一句话总结：

```text
修改之前先确认你要影响哪些行。
```

# 删

## 本质

MySQL 的“删”主要是 `delete`，本质是：

```text
从表中删除满足条件的行。
```

和 `update` 很像，`delete` 也一定要先定位数据，再执行删除。

可以这样理解：

```text
update = where 定位行 + set 修改列
delete = where 定位行 + 删除整行
```

注意：

```text
删除不是删除某个字段，而是删除整条记录。
```

比如学生表中有一行：

| id | name | age | gender | score | class_id | phone |
|---|---|---|---|---|---|---|
| 1 | 张三 | 18 | 男 | 90 | 1 | 13800138000 |

执行删除后，这一整行都会从表中消失。

## 基本删除语法

```sql
delete from 表名
where 条件;
```

例如，删除 `id = 1` 的学生：

```sql
delete from student
where id = 1;
```

含义：

```text
从 student 表中删除 id 等于 1 的那一行。
```

## 删除前一定要先查

删除操作比修改操作更危险，因为删除后数据直接没了。

所以一定要养成习惯：

```text
先 select 确认要删哪些行。
再 delete 删除。
最后 select 验证结果。
```

例如要删除 `id = 1` 的学生：

先查：

```sql
select id, name, age, score, class_id
from student
where id = 1;
```

确认无误后再删：

```sql
delete from student
where id = 1;
```

删完再查：

```sql
select id, name, age, score, class_id
from student
where id = 1;
```

如果查不到数据，说明删除成功。

## 按主键删除

最推荐的删除方式是按主键删除。

```sql
delete from student
where id = 3;
```

因为 `id` 是主键，具有唯一性，所以这条 SQL 最多删除一行。

本质：

```text
用唯一条件精确定位一条记录，然后删除。
```

## 按唯一键删除

如果某个字段有唯一约束，也可以用它删除。

比如 `phone` 是唯一键：

```sql
delete from student
where phone = '13800138008';
```

含义：

```text
删除手机号为 13800138008 的学生。
```

因为 `phone` 唯一，所以最多只会删除一行。

## 按普通条件删除

也可以按普通字段删除多行。

例如，删除 3 班所有学生：

```sql
delete from student
where class_id = 3;
```

这条 SQL 会删除所有 `class_id = 3` 的学生。

所以执行前一定要先查：

```sql
select id, name, class_id
from student
where class_id = 3;
```

确认这些人确实都要删除，再执行 `delete`。

## 删除满足多个条件的数据

例如，删除 3 班中成绩低于 60 的学生：

```sql
delete from student
where class_id = 3 and score < 60;
```

含义：

```text
只删除同时满足两个条件的学生：
1. class_id = 3
2. score < 60
```

删除前建议先查：

```sql
select id, name, score, class_id
from student
where class_id = 3 and score < 60;
```

## 配合 `in` 删除

删除 1 班和 2 班的学生：

```sql
delete from student
where class_id in (1, 2);
```

等价于：

```sql
delete from student
where class_id = 1 or class_id = 2;
```

执行前先查：

```sql
select id, name, class_id
from student
where class_id in (1, 2);
```

注意：

```text
in 适合多个确定值的匹配。
```

## 配合 `between` 删除

删除成绩在 0 到 59 之间的学生：

```sql
delete from student
where score between 0 and 59;
```

注意：

```text
between 0 and 59 是闭区间，包含 0 和 59。
```

等价于：

```sql
delete from student
where score >= 0 and score <= 59;
```

## 配合 `like` 删除

删除所有姓孙的学生：

```sql
delete from student
where name like '孙%';
```

含义：

```text
删除所有 name 以“孙”开头的学生。
```

执行前先查：

```sql
select id, name
from student
where name like '孙%';
```

## 配合 `is null` 删除

删除成绩为空的学生：

```sql
delete from student
where score is null;
```

注意不能写成：

```sql
delete from student
where score = null;
```

因为 `NULL` 不能用普通的 `=` 判断。

## 删除所有数据的危险写法

非常危险：

```sql
delete from student;
```

这条 SQL 没有 `where`，含义是：

```text
删除 student 表中的所有行。
```

注意：

```text
表结构还在，但表里的数据全部没了。
```

执行后，`student` 表仍然存在，但是变成空表。

如果只是想看表中数据，不要误写成 `delete`。

## `delete` 和 `truncate` 的区别

清空表数据有两种常见方式：

```sql
delete from student;
```

和：

```sql
truncate table student;
```

它们都可以清空表，但本质不一样。

| 操作 | 本质 | 是否可带 where | 是否逐行删除 | 自增值通常是否重置 |
|---|---|---|---|---|
| `delete` | 删除满足条件的行 | 可以 | 是 | 通常不重置 |
| `truncate` | 直接清空整张表 | 不可以 | 否 | 通常会重置 |

可以这样理解：

```text
delete 更像一行一行删数据。
truncate 更像直接把整张表清空重建。
```

所以：

```text
需要按条件删除，用 delete。
确定要清空整张表，可以用 truncate。
```

但 `truncate` 更危险，执行前一定要确认。

## `delete` 和 `drop` 的区别

```sql
delete from student;
```

含义：

```text
删除表中的数据，表结构还在。
drop table student;
```

含义：

```text
删除整张表，数据和表结构都没了。
```

对比：

| 操作 | 删除数据 | 删除表结构 |
|---|---|---|
| `delete` | 是 | 否 |
| `truncate` | 是 | 否 |
| `drop` | 是 | 是 |

一句话记忆：

```text
delete 删行。
truncate 清空表。
drop 删除表。
```

## 物理删除和逻辑删除

实际开发中，不一定真的用 `delete` 删除数据。

很多业务会使用逻辑删除。

比如给表加一个字段：

```sql
is_deleted tinyint default 0
```

含义：

```text
0 表示未删除。
1 表示已删除。
```

逻辑删除不是执行：

```sql
delete from student
where id = 1;
```

而是执行：

```sql
update student
set is_deleted = 1
where id = 1;
```

以后查询正常数据时加条件：

```sql
select id, name, score
from student
where is_deleted = 0;
```

逻辑删除的优点：

```text
1. 数据还能恢复。
2. 方便审计和追踪。
3. 避免误删造成不可逆损失。
```

物理删除的特点：

```text
真正从表中删除数据。
```

逻辑删除的特点：

```text
数据还在，只是用字段标记为已删除。
```

实际业务系统中，重要数据更常用逻辑删除。

## 删除练习

删除 `id = 3` 的学生：

```sql
select id, name, score
from student
where id = 3;

delete from student
where id = 3;

select id, name, score
from student
where id = 3;
```

删除手机号为 `13800138008` 的学生：

```sql
select id, name, phone
from student
where phone = '13800138008';

delete from student
where phone = '13800138008';

select id, name, phone
from student
where phone = '13800138008';
```

删除 3 班成绩低于 60 的学生：

```sql
select id, name, score, class_id
from student
where class_id = 3 and score < 60;

delete from student
where class_id = 3 and score < 60;

select id, name, score, class_id
from student
where class_id = 3 and score < 60;
```

删除成绩为空的学生：

```sql
select id, name, score
from student
where score is null;

delete from student
where score is null;

select id, name, score
from student
where score is null;
```

逻辑删除 `id = 5` 的学生：

```sql
update student
set is_deleted = 1
where id = 5;
```

查询未删除学生：

```sql
select id, name, score
from student
where is_deleted = 0;
```

## 删的总结

删除模板：

```sql
delete from 表名
where 条件;
```

按主键删除：

```sql
delete from student
where id = 1;
```

按条件删除多行：

```sql
delete from student
where class_id = 3;
```

清空表数据：

```sql
truncate table student;
```

删除整张表：

```sql
drop table student;
```

核心：

```text
delete = where 定位行 + 删除整行。
```

安全习惯：

```text
1. delete 前先 select。
2. delete 一定要注意 where。
3. 优先用主键或唯一键删除。
4. 删除后再 select 验证。
5. 重要业务数据优先考虑逻辑删除。
```

最危险写法：

```sql
delete from student;
```

它会删除整张表中的所有数据。

一句话总结：

```text
删除之前先确认你要删除哪些行。
```
