# 课程编号：C102003B

日期：2026  年 1 月 13  日

![image-20260113123937595](https://gitee.com/binary-whispers/pic/raw/master///20260113221450973.png)

# 《商场 VIP 消费查询系统》课程设计报告

| 学    院： |   经管   |
| :--------: | :------: |
| 指导教师： |  瞿有利  |
| 班    级： | 物流2301 |
| 学    号： | 23241023 |
| 学生姓名： |  赵文瑞  |

------

[TOC]



# 1. 需求分析

## 1.1 问题描述

设计并实现一套商场 VIP 消费查询系统，支持会员消费记录追踪、折扣积分计算等核心功能。系统需满足不同等级会员（普通 / VIP/SVIP）的差异化折扣规则，积分计算规则可灵活配置，同时保障数据持久化存储，确保系统重启后数据不丢失。

核心业务场景包括：

* 会员信息管理（增删查改）
* 消费记录录入（含折扣计算、积分累计）
* 消费明细查询（按会员 ID 统计消费总额与积分）
* 数据持久化（会员信息、交易记录存储到本地文件）

## 1.2 功能需求

### （1）会员管理功能

* 新增会员：支持录入会员号、姓名、电话，选择会员等级（普通 / VIP/SVIP），会员号唯一，入会日期自动默认为当天
* 修改会员：支持修改会员姓名、电话，保持会员号、等级、积分等核心信息不变
* 删除会员：删除会员信息的同时，同步删除该会员的所有交易记录，避免数据冗余

### （2）消费记录功能

* 支持录入消费会员号、消费日期、商品备注、消费原价
* 多态折扣计算：根据会员等级自动应用不同折扣（普通 无折扣、VIP9.5 折、SVIP8.5 折）
* 仿函数积分规则：按实付金额每满 10 元积 1 分（向下取整），积分实时累计到会员账户

### （3）查询统计功能

* 支持按会员号查询会员基本信息（ID、姓名、等级、积分、入会日期）
* 支持查询指定会员的所有消费明细，包含交易 ID、日期、商品、原价、实付、积分
* 自动统计会员累计实付金额与累计积分

### （4）数据持久化功能

* 保存功能：系统退出时自动将会员信息、交易记录写入当前工作路径(`cwd`)（文件名`members.txt/transactions.txt`）
* 读取功能：系统启动时自动加载本地文件数据，恢复上次运行状态
* 文件格式规范：采用 "|" 分隔字段，支持空格兼容与空行过滤

### （5）辅助功能

* 输入安全处理：避免回车残留、非法输入（如非数字金额、无效日期）导致程序崩溃
* 日期自动处理：支持默认当前日期，无效日期自动修正为当天
* 交易号自增：确保每次交易 ID 唯一，重启系统后不重复

## 1.3 开发环境

* 操作系统：Ubuntu
* 开发工具：VSCode
* 编程语言：C++
* 编译标准：C++98为主，少量C++11
* 依赖库：标准库（`fstream`、`vector`、`map`、`string` 等），无第三方库

## 1.4 开发过程

|  天数  |                           核心任务                           |
| :----: | :----------------------------------------------------------: |
| 第一天 | 确定课程设计题目，分析业务需求，明确核心功能与技术难点（多态、仿函数、文件持久化） |
| 第二天 | 概要设计（类结构设计、模块划分），编写基础类（`Member`、`Transaction`、`util` 工具类） |
| 第三天 | 实现核心业务逻辑（`VipSystem`类菜单功能、多态折扣、仿函数积分、文件读写），调试关键流程 |
| 第四天 |   完善边界处理（输入校验、异常场景处理），编写课程设计报告   |
| 第五天 |  系统全面测试（功能测试、数据一致性测试），提交报告与源代码  |

# 2. 概要设计

## 2.1 总体设计

系统采用面向对象设计思想，核心分为 5 大功能模块，模块间通过类接口交互：

```txt
商场VIP消费查询系统
├── 会员管理模块：处理会员增删查改功能，封装会员等级与积分逻辑
├── 交易管理模块：处理消费记录录入、折扣计算、积分累计
├── 文件持久化模块：负责会员/交易数据的文件读写与格式解析
├── 工具支撑模块：提供字符串处理、日期转换、安全输入等通用功能
└── 菜单交互模块：提供控制台菜单界面，接收用户输入并调度对应功能
```

各模块调用关系：用户通过菜单交互模块触发功能，会员 / 交易管理模块处理核心业务，工具模块提供通用支持，文件模块负责数据持久化。

## 2.2 类的定义

系统包含 6 个核心类 / 结构体，采用继承、多态、仿函数等面向对象特性，类关系如下：

### （1）抽象基类：`Member`

* 角色：定义会员通用属性与接口，实现多态特性

* 成员变量：`mId`（会员号）、`mName`（姓名）、`mPhone`（电话）、`mPoints`（积分）、`mJoinDate`（入会日期）

* 纯虚函数（多态接口）：

  * `discountRate ()`：返回会员折扣率（子类实现差异化折扣）
  * `levelName ()`：返回会员等级名称（普通 / VIP/SVIP）
  * `levelCode ()`：返回会员等级编码（0/1/2）

  

* 成员函数：``get/set` 方法、`addPoints`（积分累计）、`infoTxt`（生成文件存储字符串）

### （2）派生会员类

* `RegularMember`（普通会员）：继承 `Member`，折扣率 1.0，等级编码 0
* `VipMember`（VIP 会员）：继承 `Member`，折扣率 0.95，等级编码 1
* `SvipMember`（SVIP 会员）：继承 `Member`，折扣率 0.85，等级编码 2

### （3）核心系统类：`VipSystem`

* 角色：系统核心控制器，管理所有业务逻辑与数据存储

* 成员变量：

  * `mMembers：map<string, Member*>` 会员表（哈希映射KV模型：会员号 -> 会员对象指针）
  * `mTransactions：vector<Transaction>` 交易记录表（顺序存储所有消费记录）
  * `mNextTransactionId` 自增交易 ID（确保唯一性）
  * `mPointsCalculator：functor::PointsCalculator` 积分计算仿函数
  * `cwd`当前工作路径：文件存储路径

  

* 核心方法：

  * `run ()`：启动系统菜单循环
  * 会员管理：`addMember()/editMember()/deleteMember ()`
  * 交易管理：`recordPurchase()/queryMemberTransactions()`
  * 文件读写：`loadAll()/saveAll()/loadMembers()/saveTransactions()`

  

### （4）交易数据类：`Transaction`

* 角色：封装单条消费记录属性
* 成员变量：`transactionId`（交易 ID）、`memberId`（关联会员号）、`date`（消费日期）、`dateKey`（日期编码 yyyymmdd）、`item`（商品备注）、`amount`（原价）、`pay`（实付金额）、`pointsEarned`（本次积分）
* 成员函数：`infoTxt ()`：生成文件存储字符串

### （5）工具命名空间：util

* 角色：提供通用工具函数，解耦业务逻辑

* 核心函数：

  * `trim ()`：去除字符串首尾空格
  * `splitByPipe ()`：按 "|" 分割字符串（文件解析用）
  * `dateToInt ()`：日期字符串转编码（YYYY-MM-DD -> yyyymmdd）
  * `todayDate ()`：获取当前日期（YYYY-MM-DD）
  * 安全输入：`readLineSafe ()/readIntLine ()/readDoubleLine ()/readYesNo ()`

  

### （6）仿函数结构体：`functor::PointsCalculator`

* 角色：封装积分计算策略，支持灵活替换
* 核心逻辑：`operator ()(double pay)`：实付金额每满 10 元积 1 分，向下取整

## 2.3 接口设计

### （1）用户交互接口（菜单接口）

系统采用控制台菜单交互，用户输入对应编号触发功能，接口如下：

```
========== 商场 VIP 消费查询系统 ==========
1. 新增会员
2. 修改会员
3. 删除会员
4. 记录消费
5. 查询会员消费明细
0. 保存并退出
------------------------------------------
请输入菜单编号：
```

### （2）类间接口

* `Member` 类接口：通过纯虚函数定义统一接口，子类实现差异化逻辑，`VipSystem` 通过 `Member *` 指针调用，实现多态
* `Transaction` 类接口：通过 `infoTxt ()` 提供统一的文件存储格式，与 `loadTransactions ()` 解析逻辑一致
* `VipSystem` 类接口：对外仅暴露 `run ()` 方法，内部通过私有工具方法封装细节
* 工具接口：`util`命名空间提供静态工具函数，无状态，可直接调用
* 仿函数接口：通过 `operator ()` 重载，将积分策略封装为可调用对象，便于替换

## 2.4 运行界面设计

系统采用控制台文本界面，分为三级交互界面：

1. **主菜单界面**：展示所有功能选项，接收用户菜单编号输入

2. **功能交互界面**：

   * 新增会员：依次提示输入会员号、姓名、电话、等级
   * 修改会员：输入会员号→展示当前信息→提示输入新姓名 / 电话
   * 删除会员：输入会员号→确认删除→提示删除结果
   * 记录消费：输入会员号→提示输入消费日期、商品、原价→展示计算结果
   * 查询明细：输入会员号→展示会员信息→列出所有消费记录→展示合计数据

   

3. **反馈提示界面**：操作成功 / 失败提示（如 "会员号已存在"、"记录成功"），操作后提示 "按回车继续..."，返回主菜单

------

# 3. 详细设计

## 3.1 输入模块设计

输入模块基于 util 命名空间的安全输入函数，解决标准输入的换行残留、非法输入等问题，核心设计如下：

* **readLineSafe()**：封装 `getline`，自动去除 Windows 系统的 "\r" 换行符，避免输入残留

* **类型安全输入**：

  * `readIntLine ()`：读取整数，支持回车默认值，非法输入循环提示
  * `readDoubleLine ()`：读取浮点数（金额），支持回车默认值，非法输入循环提示
  * `readYesNo ()`：读取 y/n 确认，支持回车默认值，仅接受合法输入

  

* **输入预处理**：所有输入字符串通过 `trim ()` 去除首尾空格，避免空字符导致的逻辑错误

示例代码（安全输入整数）：

```cpp
int readIntLine(const string& prompt, int defaultValue) {
    while (true) {
        cout << prompt;
        string line;
        readLineSafe(line);
        line = trim(line);
        if (line.empty()) return defaultValue; // 回车用默认值
        istringstream iss(line);
        int v;
        if (iss >> v) return v;
        cout << "输入无效 请输入整数\n";
    }
}
```

## 3.2 会员管理模块设计（新增 / 修改 / 删除）

### 3.2.1 新增会员

* 流程：输入会员号→检查唯一性→输入姓名 / 电话→选择等级→创建会员对象→加入 mMembers 集合

* 关键逻辑：

  * 会员号唯一性校验：通过 `memberExists ()` 查询 `map` 是否存在该键
  * 等级默认值：未输入时默认 0（普通会员）
  * 入会日期：自动调用 `util::todayDate ()` 获取当前日期
  * 内存管理：通过 `createMemberByLevel ()` 创建堆对象，由 `VipSystem` 析构函数统一释放

  

核心代码片段：

```cpp
void addMember() {
    cout << "会员号：";
    string id;
    cin >> id;
    if (memberExists(id)) { cout << "该会员号已存在 \n"; return; }
    cin.ignore(1024, '\n'); // 清除换行残留
    // 输入姓名、电话...
    int levelCode = util::readIntLine("等级(0普通 1VIP 2SVIP 回车默认0)：", 0);
    mMembers[id] = createMemberByLevel(levelCode, id, name, phone, 0, util::todayDate());
}
```

### 3.2.2 修改会员

* 流程：输入会员号→查找会员对象→输入新姓名 / 电话（回车不修改）→更新对象属性
* 关键逻辑：仅允许修改姓名和电话，会员号、等级、积分等核心属性不可修改

### 3.2.3 删除会员

* 流程：输入会员号→查找会员→确认删除→删除会员对象→删除关联交易记录

* 关键逻辑：

  * 交易记录清理：遍历 `mTransactions`，保留非该会员的交易，通过 `vector.swap ()` 优化性能
  * 内存释放：先 `delete` 会员对象，再从 `map` 中 `erase` 键，避免内存泄漏

  

核心代码片段：

```cpp
vector<Transaction> remain;
for (size_t i = 0; i < mTransactions.size(); ++i) {
    if (mTransactions[i].memberId != id) remain.push_back(mTransactions[i]);
}
mTransactions.swap(remain); // 清理关联交易
delete it->second; // 释放会员对象
mMembers.erase(it);
```

## 3.3 消费记录模块设计

### 3.3.1 折扣计算（多态实现）

* 设计思路：`Member` 基类定义纯虚函数 `discountRate ()`，子类重写该函数，`VipSystem` 通过 `Member *` 指针调用，自动适配会员等级

* 差异化规则：

  * 普通会员：1.0（无折扣）
  * VIP 会员：0.95（9.5 折）
  * SVIP 会员：0.85（8.5 折）

  

* 计算逻辑：实付金额 = 原价 × 折扣率

核心代码：

```cpp
// 派生类实现
double VipMember::discountRate() const override { return 0.95; }
double SvipMember::discountRate() const override { return 0.85; }

// 消费记录中调用
double pay = amount * m->discountRate(); // 多态调用，无需判断等级
```

### 3.3.2 积分计算（仿函数实现）

* 设计思路：将积分策略封装为仿函数 `PointsCalculator`，与业务逻辑解耦，便于后续替换规则
* 积分规则：实付金额 ÷ 10，向下取整（如 68.0 元积 6 分，99.9 元积 9 分）
* 调用逻辑：通过 `mPointsCalculator (pay)` 计算积分，累计到会员账户

核心代码：

```cpp
// 仿函数定义
struct PointsCalculator {
    int operator()(double pay) const {
        return pay <= 0 ? 0 : static_cast<int>(pay / 10.0);
    }
};

// 消费记录中调用
int points = mPointsCalculator(pay);
m->addPoints(points); // 累计积分
```

### 3.3.3 交易记录存储

* 交易 ID 自增：`mNextTransactionId` 初始为 1，每次新增交易后自增，重启时通过加载历史最大 ID+1 初始化，确保唯一性
* 日期编码：`dateKey` 存储 yyyymmdd 格式整数，便于后续排序和查询

## 3.4 查询模块设计

### 3.4.1 会员消费明细查询

* 流程：输入会员号→查找会员→收集该会员所有交易→展示会员信息→遍历交易记录→统计合计金额与积分

* 关键逻辑：

  * 交易筛选：遍历 `mTransactions`，匹配 `memberId` 与输入会员号
  * 统计计算：累加实付金额`sumPay`和积分`sumPoints`
  * 格式化输出：使用 `fixed` 和 `setprecision` 确保金额保留 2 位小数

  

核心代码片段：

```cpp
vector<Transaction> list;
for (size_t i = 0; i < mTransactions.size(); ++i) {
    if (mTransactions[i].memberId == id) list.push_back(mTransactions[i]);
}
// 统计与输出...
cout << "合计：实付=" << fixed << setprecision(2) << sumPay
     << " 本次累计积分=" << sumPoints << "\n";
```

## 3.5 文件读写模块设计

### 3.5.1 文件格式规范

* 会员文件（members.txt）：字段用 "|" 分隔，格式为`id | name | phone | levelCode | points | joinDate`
* 交易文件（transactions.txt）：字段用 "|" 分隔，格式为`transactionId | memberId | date | item | amount | pay | pointsEarned`
* 兼容处理：自动过滤空行和首尾空格（trim ()）

### 3.5.2 加载数据`loadAll ()`

* 流程：先加载会员`loadMembers ()`→再加载交易`oadTransactions ()`→初始化交易号`mNextTransactionId = 最大历史 ID + 1`

* 会员加载逻辑：

  * 按行读取文件→`splitByPipe ()` 分割字段→校验字段数→创建对应等级会员对象→加入 `mMembers`

  

* 交易加载逻辑：

  * 按行读取文件→`splitByPipe ()` 分割字段→转换数据类型→构建 `Transaction` 对象→加入 `mTransactions`

  

### 3.5.3 保存数据`saveAll ()`

* 流程：调用 `saveMembers ()` 保存会员→调用 `saveTransactions ()` 保存交易
* 保存逻辑：遍历会员 / 交易集合，调用 `infoTxt ()` 生成标准化字符串，写入文件
* 一致性保障：会员与交易的存储格式与加载格式严格一致，确保数据完整性

核心代码（保存会员）：

```cpp
void saveMembers() const {
    ofstream fout(mMemberFilePath.c_str());
    if (!fout) return;
    for (auto it = mMembers.begin(); it != mMembers.end(); ++it) {
        fout << it->second->infoTxt() << "\n"; // 统一格式输出
    }
}
```

## 3.6 工具模块设计

util 命名空间提供 4 类核心工具函数，支撑系统各模块：

1. **字符串处理**：`trim ()`（去空格）、`splitByPipe ()`（按 "|" 分割）→ 用于文件解析
2. **日期处理**：`todayDate ()`（获取当前日期）、`dateToInt ()`（日期转编码）→ 用于交易日期管理
3. **安全输入**：`readLineSafe ()`、`readIntLine ()` 等→ 用于用户交互输入
4. **确认交互**：`readYesNo ()`→ 用于删除等危险操作的确认

------

# 4. 测试分析

```bash
vect@VM-0-11-ubuntu:~/CppRepo/VIPSystem/src$ ls
functors.h  main.cpp  member.h  transaction.h  utility.h  vip_system.h
vect@VM-0-11-ubuntu:~/CppRepo/VIPSystem/src$ g++ -o proc main.cpp 
vect@VM-0-11-ubuntu:~/CppRepo/VIPSystem/src$ ./proc 

========== 商场 VIP 消费查询系统 ==========
1. 新增会员
2. 修改会员
3. 删除会员
4. 记录消费
5. 查询会员消费明细
0. 保存并退出
------------------------------------------
请输入菜单编号：1

[新增会员]
会员号：001
姓名：张三
电话：123456
等级(0普通 1VIP 2SVIP 回车默认0)：
新增成功 
会员号=001 姓名=张三 电话=123456 等级=普通 积分=0 入会=2026-01-13
按回车继续...


========== 商场 VIP 消费查询系统 ==========
1. 新增会员
2. 修改会员
3. 删除会员
4. 记录消费
5. 查询会员消费明细
0. 保存并退出
------------------------------------------
请输入菜单编号：4

[记录消费]
会员号：001
会员等级=普通 折扣=1.00
消费日期(YYYY-MM-DD 回车默认今天)：
商品/备注(回车默认未填写)：电动车
原价金额：2688
记录成功：实付=2688.00 积分+268 当前积分=268
按回车继续...


========== 商场 VIP 消费查询系统 ==========
1. 新增会员
2. 修改会员
3. 删除会员
4. 记录消费
5. 查询会员消费明细
0. 保存并退出
------------------------------------------
请输入菜单编号：5

[查询会员消费明细]
会员号：001
会员信息：
会员号=001 姓名=张三 电话=123456 等级=普通 积分=268 入会=2026-01-13
消费记录：
交易#1 日期=2026-01-13 商品=电动车 原价=2688.00 实付=2688.00 积分+268
合计：实付=2688.00 本次累计积分=268
按回车继续...

========== 商场 VIP 消费查询系统 ==========
1. 新增会员
2. 修改会员
3. 删除会员
4. 记录消费
5. 查询会员消费明细
0. 保存并退出
------------------------------------------
请输入菜单编号：0
已保存，退出 
vect@VM-0-11-ubuntu:~/CppRepo/VIPSystem/src$ ls
functors.h  member.h     proc           transactions.txt  vip_system.h
main.cpp    members.txt  transaction.h  utility.h
vect@VM-0-11-ubuntu:~/CppRepo/VIPSystem/src$ cat *.txt
001 | 张三 | 123456 | 0 | 268 | 2026-01-13
1 | 001 | 2026-01-13 | 电动车 | 2688.00 | 2688.00 | 268
vect@VM-0-11-ubuntu:~/CppRepo/VIPSystem/src$ 
```

说明：命令行演示了增删查改的基本功能，并且程序实现了持久化，将会员信息和消费记录持久化保存到当前工作路径

运行截图：

![image-20260113215313815](https://gitee.com/binary-whispers/pic/raw/master///20260113215316075.png)

![image-20260113215339731](https://gitee.com/binary-whispers/pic/raw/master///20260113215346885.png)

# 5. 课程设计总结

本学期通过C++的课程学习，体会到了面向对象的三大特性：**封装、继承、多态**，以及各种容器的使用，尤其是对于封装的特性，多个容器使用同一套迭代器的逻辑，底层的数据结构不同，上层暴露的用户接口操作却都相同，这种思想和设计方式太伟大了。最近正在学习OS知识，OS的设计也有封装和多态的特性，我想，接下来的学习，我应该学习一些C++的新特性，理解C++只是工具，底层的数据结构、计算机组成原理、操作系统、数据库、计算机网络才是核心，然后学习多线程编程和网络编程，以及补充底层知识，持续输出自己的CSDN和GitHub，不断学习，无限进步！

------

# 6. 附录：程序文件清单

|      文件名称      |                           功能描述                           |
| :----------------: | :----------------------------------------------------------: |
|   `vip_system.h`   |     系统核心类 `VipSystem` 定义，包含菜单功能和业务逻辑      |
|     `member.h`     | 会员基类 `Member` 及派生类（Regular/VIP/SVIP）定义，工厂函数声明 |
|  `transaction.h`   |    交易类 `Transaction` 定义，包含交易属性和文件存储方法     |
|    `functors.h`    |       仿函数 `PointsCalculator` 定义，封装积分计算策略       |
|    `utility.h`     | 工具命名空间 `util` 定义，包含字符串、日期、安全输入等工具函数 |
|     `main.cpp`     |          程序入口，创建 `VipSystem `对象并启动系统           |
|   `members.txt`    |                会员数据持久化文件（自动生成）                |
| `transactions.txt` |                交易数据持久化文件（自动生成）                |

``` cpp
// functors.h
#pragma once

/*
 * 仿函数（Functor）
 * - PointsCalculator：积分策略（按实付每满10元积1分）
 */
namespace functor {

    struct PointsCalculator {
        int operator()(double pay) const {
            if (pay <= 0) return 0;
            // static_cast<int> 向下取整 68.0->6
            return static_cast<int>(pay / 10.0);
        }
    };

} 

```

```cpp
// member.h
#pragma once
#include <string>
#include <sstream>
#include <iostream>

using namespace std;

/*
 * 会员体系（继承 + 动态）
 * - Member：抽象基类（虚函数：折扣/等级）
 * - Regular/VIP/SVIP：派生类
 */

class Member {
protected:
    string mId;
    string mName;
    string mPhone;
    int    mPoints;
    string mJoinDate;

public:
    Member() : mPoints(0) {}

    Member(const string& id, const string& name, const string& phone,
           int points, const string& joinDate)
        : mId(id), mName(name), mPhone(phone), mPoints(points), mJoinDate(joinDate) {}
    
    //必须vir析构 保证delete子类正常
    virtual ~Member() {}

    // 多态接口 不同等级返回不同折扣/名称
    // 只拿Member指针 运行时自动分配到子类
    virtual double discountRate() const = 0;
    virtual string levelName() const = 0;
    virtual int    levelCode() const = 0;

    // get函数
    const string& getId() const { return mId; }
    const string& getName() const { return mName; }
    const string& getPhone() const { return mPhone; }
    int getPoints() const { return mPoints; }
    const string& getJoinDate() const { return mJoinDate; }

    // set函数
    void setName(const string& name) { mName = name; }
    void setPhone(const string& phone) { mPhone = phone; }

    void addPoints(int delta) {
        mPoints += delta;
        if (mPoints < 0) mPoints = 0;   // 积分不可能为负数
    }

    // 写入文件 统一用 | 分隔 和读取逻辑完全一致 
    string infoTxt() const {
        ostringstream oss;
        oss << mId << " | " << mName << " | " << mPhone << " | "
            << levelCode() << " | " << mPoints << " | " << mJoinDate;
        return oss.str();
    }
};

class RegularMember : public Member {
public:
    using Member::Member;
    double discountRate() const override { return 1.0; }
    string levelName() const override { return "普通"; }
    int levelCode() const override { return 0; }
};

class VipMember : public Member {
public:
    using Member::Member;
    double discountRate() const override { return 0.95; }
    string levelName() const override { return "VIP"; }
    int levelCode() const override { return 1; }
};

class SvipMember : public Member {
public:
    using Member::Member;
    double discountRate() const override { return 0.85; }
    string levelName() const override { return "SVIP"; }
    int levelCode() const override { return 2; }
};

/*
 * 从文件读 levelCode 后创建对应对象
 * 返回 new 出来的指针 由 VipSystem 统一 delete（避免内存泄漏）
 */
inline Member* createMemberByLevel(int levelCode,
                                  const string& id,
                                  const string& name,
                                  const string& phone,
                                  int points,
                                  const string& joinDate) {
    if (levelCode == 1) return new VipMember(id, name, phone, points, joinDate);
    if (levelCode == 2) return new SvipMember(id, name, phone, points, joinDate);
    return new RegularMember(id, name, phone, points, joinDate);
}

```

```cpp
// transaction.h
#pragma once
#include <string>
#include <sstream>
#include <iomanip>

using namespace std;

/*
 * 消费记录 文件存储 + 查询展示
 * - dateKey：yyyymmdd 
 */

class Transaction {
public:
    long   transactionId;
    string memberId;
    string date;
    int    dateKey;
    string item;
    double amount;
    double pay;
    int    pointsEarned;

public:
    Transaction()
        : transactionId(0), dateKey(0), amount(0.0), pay(0.0), pointsEarned(0) {}

    // transactionId | memberId | date | item | amount | pay | pointsEarned
    string infoTxt() const {
        ostringstream oss;
        oss << transactionId << " | "
            << memberId << " | "
            << date << " | "
            << item << " | "
            << fixed << setprecision(2) << amount << " | "
            << fixed << setprecision(2) << pay << " | "
            << pointsEarned;
        return oss.str();
    }
};

```

```cpp
//utility.h

#pragma once
#include <string>
#include <vector>
#include <sstream>
#include <iostream>
#include <ctime>
#include <cstdlib>
#include <cstdio>

using namespace std;

/*
 * 工具函数（尽量保持简单）
 * - trim / splitByPipe：文件解析（members.txt / transactions.txt）
 * - dateToInt：把 YYYY-MM-DD -> yyyymmdd（用于存储/比较）
 * - readLineSafe：getline 安全读取
 * - readIntLine / readDoubleLine / readYesNo：交互输入（回车默认）
 *
 */

namespace util {

    inline string trim(const string& s) {
        size_t b = 0;
        while (b < s.size() && (s[b] == ' ' || s[b] == '\t' || s[b] == '\n' || s[b] == '\r')) ++b;
        size_t e = s.size();
        while (e > b && (s[e - 1] == ' ' || s[e - 1] == '\t' || s[e - 1] == '\n' || s[e - 1] == '\r')) --e;
        return s.substr(b, e - b);
    }

    inline vector<string> splitByPipe(const string& line) {
        // 文件行格式：字段之间用 | 分隔
        // 字段两侧可能有空格 所以 push 之前必须 trim
        vector<string> parts;
        string buf;
        for (size_t i = 0; i < line.size(); ++i) {
            if (line[i] == '|') {
                parts.push_back(trim(buf));
                buf.clear();
            } else {
                buf += line[i];
            }
        }
        parts.push_back(trim(buf));
        return parts;
    }

    // YYYY-MM-DD -> yyyymmdd
    //  year*10000 *1000 好像都有点问题 todo
    inline int dateToInt(const string& date) {
        // date 来自用户输入或文件 可能为空
        if (date.size() != 10) return 0;
        if (date[4] != '-' || date[7] != '-') return 0; 

        int y = atoi(date.substr(0, 4).c_str());
        int m = atoi(date.substr(5, 2).c_str());
        int d = atoi(date.substr(8, 2).c_str());

        // 最小范围校验 没有加闰年
        if (m < 1 || m > 12) return 0;
        if (d < 1 || d > 31) return 0;

        return y * 10000 + m * 100 + d;
    }

    inline string todayDate() {
        // 统一日期输出格式：YYYY-MM-DD
        time_t now = time(nullptr);
        tm* lt = localtime(&now);
        char buf[32];
        sprintf(buf, "%04d-%02d-%02d", lt->tm_year + 1900, lt->tm_mon + 1, lt->tm_mday);
        return string(buf);
    }

    inline void readLineSafe(string& out) {
        // 安全读取一整行 并去掉 Windows 的 '\r'
        getline(cin, out);
        if (!out.empty() && out.back() == '\r') out.pop_back(); 
    }

    inline int readIntLine(const string& prompt, int defaultValue) {
        // 用整行读取 允许用户直接回车用默认值
        while (true) {
            cout << prompt;
            string line;
            readLineSafe(line);
            line = trim(line);
            if (line.empty()) return defaultValue;

            istringstream iss(line);
            int v;
            if (iss >> v) return v;
            cout << "输入无效 请输入整数\n";
        }
    }

    inline double readDoubleLine(const string& prompt, double defaultValue) {
        // 整行读取 + 回车默认
        while (true) {
            cout << prompt;
            string line;
            readLineSafe(line);
            line = trim(line);
            if (line.empty()) return defaultValue;

            istringstream iss(line);
            double v;
            if (iss >> v) return v;
            cout << "输入无效 请输入数字\n";
        }
    }

    inline bool readYesNo(const string& prompt, bool defaultValue) {
        // 允许回车默认
        while (true) {
            cout << prompt;
            string line;
            readLineSafe(line);
            line = trim(line);

            if (line.empty()) return defaultValue;
            if (line == "y" || line == "Y") return true;
            if (line == "n" || line == "N") return false;

            cout << "请输入 y 或 n（或回车默认）\n";
        }
    }

} 

```

```cpp
// vip_system.h
#pragma once
#include <fstream>
#include <map>
#include <vector>
#include <string>
#include <iostream>
#include <iomanip>
#include <cstdlib>

#include "member.h"
#include "transaction.h"
#include "functors.h"
#include "utility.h"

using namespace std;

/*
 * VipSystem：系统核心
 * - mMembers：会员表（memberId -> Member*）
 * - mTransactions：交易记录表（顺序存储）
 * - 文件读写：members.txt / transactions.txt
 *
 * 菜单：
 * 1 新增会员
 * 2 修改会员
 * 3 删除会员（同时删除其交易记录，减少复杂分支）
 * 4 记录消费（多态折扣 + 仿函数积分）
 * 5 查询会员消费明细
 * 0 保存并退出
 * 
 * 设计：
 * 1) 多态：Member* 调用 virtual discountRate()/levelName() 实现不同等级折扣 
 * 2) 仿函数：PointsCalculator 把“积分策略”独立出来，展示可替换策略 
 * 3) 文件持久化：Member/Transaction 各自提供 infoTxt()，VipSystem 负责读写与对象生命周期 
 *
 */

class VipSystem {
private:
    map<string, Member*> mMembers;
    vector<Transaction>  mTransactions;

    long mNextTransactionId = 1;
    functor::PointsCalculator mPointsCalculator;

    string mMemberFilePath;
    string mTransactionFilePath;

private:
    // 防止浅拷贝导致重复释放
    VipSystem(const VipSystem&);
    VipSystem& operator=(const VipSystem&);

public:
    VipSystem(const string& memberFilePath, const string& transactionFilePath)
        : mMemberFilePath(memberFilePath)
        , mTransactionFilePath(transactionFilePath) {}

    ~VipSystem() {
        clearMembers(); // 统一释放 Member*
    }

public:
    void run() {
        loadAll();

        while (true) {
            cout << "\n========== 商场 VIP 消费查询系统 ==========\n";
            cout << "1. 新增会员\n";
            cout << "2. 修改会员\n";
            cout << "3. 删除会员\n";
            cout << "4. 记录消费\n";
            cout << "5. 查询会员消费明细\n";
            cout << "0. 保存并退出\n";
            cout << "------------------------------------------\n";
            cout << "请输入菜单编号：";

            int menu;
            cin >> menu;

            switch (menu) {
                case 1: addMember(); break;
                case 2: editMember(); break;
                case 3: deleteMember(); break;
                case 4: recordPurchase(); break;
                case 5: queryMemberTransactions(); break;
                case 0:
                    saveAll();
                    cout << "已保存，退出 \n";
                    return;
                default:
                    cout << "无效选项 \n";
                    break;
            }

            // cin >> 之后会残留 '\n' 用 ignore 等待回车
            cout << "按回车继续...";
            cin.ignore(1024, '\n');
            string dummy;
            util::readLineSafe(dummy);
        }
    }

private:
    // ================== 内部工具 ==================

    void clearMembers() {
        // 释放 new 出来的 Member*
        for (auto it = mMembers.begin(); it != mMembers.end(); ++it) {
            delete it->second;
            it->second = nullptr;
        }
        mMembers.clear();
    }

    Member* findMember(const string& memberId) const {
        auto it = mMembers.find(memberId);
        if (it == mMembers.end()) return nullptr;
        return it->second;
    }

    bool memberExists(const string& memberId) const {
        return mMembers.find(memberId) != mMembers.end();
    }

    void printMemberSimple(const Member* m) const {
        if (!m) return;
        cout << "会员号=" << m->getId()
             << " 姓名=" << m->getName()
             << " 电话=" << m->getPhone()
             << " 等级=" << m->levelName()
             << " 积分=" << m->getPoints()
             << " 入会=" << m->getJoinDate()
             << "\n";
    }

    void printTransactionSimple(const Transaction& t) const {
        cout << "交易#" << t.transactionId
             << " 日期=" << t.date
             << " 商品=" << t.item
             << " 原价=" << fixed << setprecision(2) << t.amount
             << " 实付=" << fixed << setprecision(2) << t.pay
             << " 积分+" << t.pointsEarned
             << "\n";
    }

    // ================== 文件读写 ==================

    void loadMembers() {
        
        ifstream fin(mMemberFilePath.c_str());
        if (!fin) return; // 第一次运行没有文件很正常

        string line;
        while (getline(fin, line)) {
            line = util::trim(line);
            if (line.empty()) continue;

            // id | name | phone | level | points | joinDate
            vector<string> p = util::splitByPipe(line);
            if (p.size() < 6) continue;

            string id = p[0];
            if (id.empty()) continue;

            string name = p[1];
            string phone = p[2];
            int levelCode = atoi(p[3].c_str());
            int points = atoi(p[4].c_str());
            string joinDate = p[5];

            // 根据 levelCode 创建不同子类对象
            mMembers[id] = createMemberByLevel(levelCode, id, name, phone, points, joinDate);
        }
        fin.close();
    }

    void loadTransactions() {
        mTransactions.clear(); // 避免重复叠加

        ifstream fin(mTransactionFilePath.c_str());
        if (!fin) return;

        string line;
        while (getline(fin, line)) {
            line = util::trim(line);
            if (line.empty()) continue;

            // transactionId | memberId | date | item | amount | pay | pointsEarned
            vector<string> p = util::splitByPipe(line);
            if (p.size() < 7) continue;

            Transaction t;
            t.transactionId = atol(p[0].c_str());
            t.memberId = p[1];
            t.date = p[2];
            t.dateKey = util::dateToInt(t.date); 
            t.item = p[3];
            t.amount = atof(p[4].c_str());
            t.pay = atof(p[5].c_str());
            t.pointsEarned = atoi(p[6].c_str());

            mTransactions.push_back(t);
        }
        fin.close();
    }

    void saveMembers() const {
        ofstream fout(mMemberFilePath.c_str());
        if (!fout) return;

        for (auto it = mMembers.begin(); it != mMembers.end(); ++it) {
            // infoTxt() 的字段顺序必须和 loadMembers() 解析一致
            fout << it->second->infoTxt() << "\n";
        }
    }

    void saveTransactions() const {
        ofstream fout(mTransactionFilePath.c_str());
        if (!fout) return;

        for (size_t i = 0; i < mTransactions.size(); ++i) {
            fout << mTransactions[i].infoTxt() << "\n";
        }
    }

    void loadAll() {
        loadMembers();
        loadTransactions();

        // 自增交易号 避免程序重启后交易号重复
        long maxId = 0;
        for (size_t i = 0; i < mTransactions.size(); ++i) {
            if (mTransactions[i].transactionId > maxId) maxId = mTransactions[i].transactionId;
        }
        mNextTransactionId = maxId + 1;
    }

    void saveAll() const {
        saveMembers();
        saveTransactions();
    }

    // ================== 菜单功能 ==================

    void addMember() {
        cout << "\n[新增会员]\n";
        cout << "会员号：";
        string id;
        cin >> id;

        if (id.empty()) { cout << "会员号不能为空 \n"; return; }
        if (memberExists(id)) { cout << "该会员号已存在 \n"; return; }

        // 下面要 getline 先清掉 cin >> 的换行
        cin.ignore(1024, '\n');

        cout << "姓名：";
        string name; util::readLineSafe(name); name = util::trim(name);

        cout << "电话：";
        string phone; util::readLineSafe(phone); phone = util::trim(phone);

        int levelCode = util::readIntLine("等级(0普通 1VIP 2SVIP 回车默认0)：", 0);
        if (levelCode < 0 || levelCode > 2) levelCode = 0;

        string joinDate = util::todayDate();
        mMembers[id] = createMemberByLevel(levelCode, id, name, phone, 0, joinDate);

        cout << "新增成功 \n";
        printMemberSimple(mMembers[id]);
    }

    void editMember() {
        cout << "\n[修改会员]\n";
        cout << "会员号：";
        string id;
        cin >> id;

        Member* m = findMember(id);
        if (!m) { cout << "未找到该会员 \n"; return; }

        cout << "当前信息：\n";
        printMemberSimple(m);

        cin.ignore(1024, '\n');

        cout << "新姓名(回车不改)：";
        string name; util::readLineSafe(name); name = util::trim(name);
        if (!name.empty()) m->setName(name);

        cout << "新电话(回车不改)：";
        string phone; util::readLineSafe(phone); phone = util::trim(phone);
        if (!phone.empty()) m->setPhone(phone);

        cout << "修改完成：\n";
        printMemberSimple(m);
    }

    void deleteMember() {
        cout << "\n[删除会员]\n";
        cout << "会员号：";
        string id;
        cin >> id;

        auto it = mMembers.find(id);
        if (it == mMembers.end()) { cout << "未找到该会员 \n"; return; }

        cout << "将删除：\n";
        printMemberSimple(it->second);

        if (!util::readYesNo("确认删除？(y/n，回车默认n)：", false)) {
            cout << "已取消 \n";
            return;
        }

        // 删除会员时顺带删除其交易记录 避免孤儿交易->类比孤儿进程
        vector<Transaction> remain;
        remain.reserve(mTransactions.size());
        for (size_t i = 0; i < mTransactions.size(); ++i) {
            if (mTransactions[i].memberId != id) remain.push_back(mTransactions[i]);
        }
        mTransactions.swap(remain);

        // 先 delete 再 erase
        delete it->second;
        mMembers.erase(it);

        cout << "删除成功（含该会员交易记录） \n";
    }

    void recordPurchase() {
        cout << "\n[记录消费]\n";
        cout << "会员号：";
        string id;
        cin >> id;

        Member* m = findMember(id);
        if (!m) { cout << "未找到该会员 \n"; return; }

        cout << "会员等级=" << m->levelName()
             << " 折扣=" << fixed << setprecision(2) << m->discountRate()
             << "\n";

        cin.ignore(1024, '\n');

        cout << "消费日期(YYYY-MM-DD 回车默认今天)：";
        string date; util::readLineSafe(date); date = util::trim(date);
        if (date.empty()) date = util::todayDate();

        // 日期不合法时 直接回退到今天
        if (util::dateToInt(date) == 0) {
            cout << "日期格式不合法，已自动改为今天 \n";
            date = util::todayDate();
        }

        cout << "商品/备注(回车默认未填写)：";
        string item; util::readLineSafe(item); item = util::trim(item);
        if (item.empty()) item = "未填写";

        double amount = util::readDoubleLine("原价金额：", 0.0);
        if (amount < 0) amount = 0.0;

        // 不同等级折扣不同
        double pay = amount * m->discountRate();

        // 仿函数积分策略
        int points = mPointsCalculator(pay);

        Transaction t;
        t.transactionId = mNextTransactionId++;
        t.memberId = id;
        t.date = date;
        t.dateKey = util::dateToInt(date);
        t.item = item;
        t.amount = amount;
        t.pay = pay;
        t.pointsEarned = points;

        mTransactions.push_back(t);
        m->addPoints(points);

        cout << "记录成功：实付=" << fixed << setprecision(2) << pay
             << " 积分+" << points
             << " 当前积分=" << m->getPoints()
             << "\n";
    }

    void queryMemberTransactions() {
        cout << "\n[查询会员消费明细]\n";
        cout << "会员号：";
        string id;
        cin >> id;

        Member* m = findMember(id);
        if (!m) { cout << "未找到该会员 \n"; return; }

        cout << "会员信息：\n";
        printMemberSimple(m);

        // 收集该会员交易
        vector<Transaction> list;
        for (size_t i = 0; i < mTransactions.size(); ++i) {
            if (mTransactions[i].memberId == id) list.push_back(mTransactions[i]);
        }

        if (list.empty()) {
            cout << "暂无消费记录 \n";
            return;
        }

        cout << "消费记录：\n";
        double sumPay = 0.0;
        int sumPoints = 0;

        for (size_t i = 0; i < list.size(); ++i) {
            printTransactionSimple(list[i]);
            sumPay += list[i].pay;
            sumPoints += list[i].pointsEarned;
        }

        cout << "合计：实付=" << fixed << setprecision(2) << sumPay
             << " 本次累计积分=" << sumPoints
             << "\n";
    }
};

```

```cpp
// main.cpp
#include "vip_system.h"

int main() {
    VipSystem system("members.txt", "transactions.txt");
    system.run();
    return 0;
}

```

