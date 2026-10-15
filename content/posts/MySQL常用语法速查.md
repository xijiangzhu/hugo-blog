---
title: MySQL常用语法速查
date: 2026-10-12 10:00:00
tags:
  - MySQL
  - SQL
  - 数据库
categories:
  - 技术
  - 数据库
description: 整理 MySQL 常用语法:数据库、表、表结构、索引、视图、常用函数、存储过程与触发器的创建、查看、修改和删除。
---

本文整理 MySQL 日常开发和运维中最常用的语法,包括数据库、表、表结构、索引、视图、函数、存储过程和触发器,方便随时查阅。

<!-- more -->

## 一、数据库

```sql
-- 创建一个数据库
create database k default character set utf8 collate utf8_general_ci;

-- 删除一个数据库
drop database if exists k;
```

> 提示:`utf8` 在 MySQL 中实际只支持 3 字节字符,无法存储 emoji 等 4 字节字符。新项目建议使用 `utf8mb4`:
>
> ```sql
> create database k default character set utf8mb4 collate utf8mb4_general_ci;
> ```

## 二、表

### 1. 查看所有的表

```sql
show tables;
```

### 2. 创建表

```sql
-- 简单创建
create table n(id int, name varchar(10));

-- 带主键、外键、唯一约束
create table m(
    id int,
    name varchar(10),
    primary key (id),
    foreign key (id) references n(id),
    unique (name)
);

-- 不带约束
create table m(id int, name varchar(10));
```

> 上面三条是不同写法的示例,同名的表不能重复创建。

### 3. 通过已有表创建新表

```sql
-- 直接将查询结果导入或复制到新创建的表(复制结构和数据,不复制索引)
create table n select * from m;

-- 新表与已存在的表数据结构相同(只复制结构,不复制数据)
create table m like n;
```

### 4. 创建临时表

临时表在当前连接期间存在,断开连接时 MySQL 会自动删除表并释放空间,也可以手动删除。

```sql
create temporary table l(id int, name varchar(10));

-- 直接将查询结果导入新创建的临时表
create temporary table tt select * from n;
```

### 5. 删除表

```sql
drop table if exists m;
```

### 6. 重命名表

```sql
alter table n rename m;
rename table n to m;
```

### 7. 查看表结构

以下五条语句效果相同:

```sql
desc n;              -- 因为简单,所以建议使用
describe n;
show columns in n;
show columns from n;
explain n;
```

### 8. 查看表的创建语句

```sql
show create table n;
```

## 三、表结构

```sql
-- 添加字段
alter table n add age varchar(2);

-- 删除字段
alter table n drop age;

-- 同时更改字段名和属性(把 age 改名为 a,类型改为 int)
alter table n change age a int;

-- 只更改字段属性(不改名)
alter table n modify age varchar(7);
```

## 四、索引

### 1. 主键

```sql
-- 添加主键
alter table n add primary key (id);
alter table n add constraint pk_n primary key (id);  -- 主键只有一个,所以定义键名意义不大

-- 删除主键
alter table n drop primary key;
```

### 2. 外键

```sql
-- 添加外键
alter table m add foreign key (id) references n(id);                          -- 自动生成键名 m_ibfk_1
alter table m add constraint fk_id foreign key (id) references n(id);         -- 使用自定义的键名 fk_id

-- 删除外键
alter table m drop foreign key fk_id;

-- 修改外键(先删除再重建)
alter table m drop foreign key `fk_id`, add constraint fk_id2 foreign key (id) references n(id);
```

### 3. 唯一键

```sql
alter table n add unique (name);
alter table n add unique u_name (name);
alter table n add unique index u_name (name);
alter table n add constraint u_name unique (name);
create unique index u_name on n(name);
```

> 以上是同一个功能的不同写法,选一种即可。

### 4. 普通索引

```sql
alter table n add index (age);
alter table n add index i_age (age);
create index i_age on n(age);
```

### 5. 删除索引或唯一键

```sql
alter table n drop index index_name;
drop index u_name on n;
drop index i_age on n;
```

## 五、视图

```sql
-- 创建视图
create view v as select id, name from n;
create view v(id, name) as select id, name from n;

-- 查看视图(与表操作类似)
select * from v;
desc v;

-- 查看创建视图的语句
show create view v;

-- 更改视图
create or replace view v as select name, age from n;
alter view v as select name from n;

-- 删除视图
drop view if exists v;
```

## 六、函数

### 1. 聚合函数

```sql
select count(id) as total from n;    -- 总数
select sum(age) as all_age from n;   -- 总和
select avg(age) as all_age from n;   -- 平均值
select max(age) as all_age from n;   -- 最大值
select min(age) as all_age from n;   -- 最小值
```

### 2. 数学函数

```sql
select abs(-5);                      -- 绝对值:5
select bin(15), oct(15), hex(15);    -- 二进制、八进制、十六进制:1111  17  F
select rand(5);                      -- 传入种子参数后,每次生成相同的随机数序列
select round(1415.1415);             -- 四舍五入:1415
select round(1415.1415, 3);          -- 四舍五入保留三位小数:1415.142
select round(1415.1415, -1);         -- 四舍五入到十位:1420
select truncate(1415.1415, 3);       -- 截断保留三位小数:1415.141
select truncate(1415.1415, -1);      -- 截断到十位:1410
```

### 3. 字符串函数

```sql
select concat('a', 'p', 'p', 'le');          -- 连接字符串:apple
select concat_ws(',', 'a', 'p', 'p', 'le');  -- 用 ',' 连接字符串:a,p,p,le
select insert('chinese', 3, 2, 'in');        -- 从第 3 个位置开始,把 2 个字符替换为 'in':chinese
select left('chinese', 4);                   -- 左边 4 个字符:chin
select right('chinese', 3);                  -- 右边 3 个字符:ese
select substring('chinese', 3);              -- 从第 3 个字符开始到结尾:inese
select substring('chinese', -3);             -- 从倒数第 3 个字符开始到结尾:ese
select substring('chinese', 3, 2);           -- 从第 3 个字符开始取 2 个字符:in
select trim(' chinese ');                    -- 去掉两边的空格:'chinese'
select ltrim(' chinese ');                   -- 去掉左边的空格:'chinese '
select rtrim(' chinese ');                   -- 去掉右边的空格:' chinese'
select repeat('boy', 3);                     -- 重复字符串三次:'boyboyboy'
select reverse('chinese');                   -- 反转字符串:'esenihc'
select length('chinese');                    -- 字符串长度:7
select upper('chinese'), lower('chinese');   -- 转大写、转小写:CHINESE  chinese
select ucase('chinese'), lcase('chinese');   -- 同上:CHINESE  chinese
select position('i' in 'chinese');           -- 'i' 第一次出现的位置:3
select position('e' in 'chinese');           -- 'e' 第一次出现的位置:5
select strcmp('abc', 'abd');                 -- 比较字符串,第一个小于第二个返回负数:-1
select strcmp('abc', 'abb');                 -- 第一个大于第二个返回正数:1
```

### 4. 时间函数

```sql
select current_date, current_time, now();
-- 2018-01-13  12:33:43  2018-01-13 12:33:43

select hour(current_time), minute(current_time), second(current_time);
-- 12  31  34

select year(current_date), month(current_date), week(current_date);
-- 2018  1  1

select quarter(current_date);
-- 1

select monthname(current_date), dayname(current_date);
-- January  Saturday

select dayofweek(current_date), dayofmonth(current_date), dayofyear(current_date);
-- 7  13  13
```

> 以上结果是以 2018-01-13 为例,实际结果取决于执行时的时间。

### 5. 控制流函数

```sql
select if(3>2, 't', 'f'), if(3<2, 't', 'f');   -- t  f
select ifnull(null, 't'), ifnull(2, 't');      -- t  2
select isnull(1), isnull(1/0);                 -- 0  1  (是 null 返回 1,不是 null 返回 0)
select nullif('a', 'a'), nullif('a', 'b');     -- null  a  (两个参数相等返回 null,否则返回第一个参数)

select case 2
    when 1 then 'first'
    when 2 then 'second'
    when 3 then 'third'
    else 'other'
end;
-- second
```

### 6. CASE 的几种常见用法

**用法一**:`case 字段 when 值`

```sql
select
    case c_sex when 1 then '男'
    else '女'
    end as 性别,
    count(*) as 人数
from user_info
group by 性别;
```

**用法二**:`case when 条件`

```sql
select
    case when c_sex = 1 then '男'
    else '女'
    end as 性别,
    count(*) as 人数
from user_info
group by 性别;
```

**用法三**:部分处理,其他保留原样

```sql
select
    case when c_quantity is null then '0'
    else c_quantity
    end as 用量,
    count(*)
from material_detail
group by 用量;
```

**用法四**:配合 `between` 做区间分组

```sql
select
    case when c_month between 1 and 3 then '一'
         when c_month between 4 and 6 then '二'
         when c_month between 7 and 9 then '三'
         else '四'
    end as 季度,
    sum(c_salesum) as 销量
from spiderdata_car_salesum
group by 季度
order by 季度;
```

> `group by` 后面要写别名本身(如 `性别`),**不要加单引号**。加了单引号会被当成固定的字符串常量,导致分组结果不正确。

### 7. 系统信息函数

```sql
select database();       -- 当前数据库名:test
select connection_id();  -- 当前连接 id:306
select user();           -- 当前用户:root@localhost
select version();        -- 当前 MySQL 版本
select found_rows();     -- 上次查询检索到的行数
```

## 七、存储过程

### 1. 创建存储过程

创建前需要用 `delimiter` 修改语句结束符,避免过程体里的分号提前结束语句。注意 `delimiter` 命令所在行**不要写注释**。

```sql
delimiter //

-- 无参数
create procedure getdates()
begin
    select * from test;
end //

-- in 参数(输入)
create procedure getdates_2(in id int)
begin
    select * from test where a = id;
end //

-- out 参数(输出)
create procedure getdates_3(out sum int)
begin
    set sum = (select count(*) from test);
end //

-- inout 参数(输入输出)
create procedure getdates_4(inout i int)
begin
    set i = i + 1;
end //

delimiter ;
```

### 2. 删除存储过程

```sql
drop procedure if exists getdates;
```

### 3. 修改存储过程的特性

```sql
alter procedure getdates modifies sql data;
```

> `alter procedure` 只能修改特性,不能修改过程体。要修改过程体,需要先删除再重建。

### 4. 查看存储过程

```sql
show procedure status like 'getdates';   -- 查看状态
show create procedure getdates_3;        -- 查看创建语句
```

### 5. 调用存储过程

```sql
call getdates();
call getdates_2(1);

call getdates_3(@s);
select @s;

set @i = 1;
call getdates_4(@i);
select @i;   -- 结果为 2
```

## 八、触发器

### 1. 语法

```sql
create trigger 触发器名
{before | after}                 -- 触发时间
{insert | update | delete}       -- 触发条件
on 表名
for each row
begin
    -- 触发事件(要执行的语句)
end;
```

> 语法中的 `{ }` 和 `|` 表示"二选一",实际书写时只保留其中一项。

### 2. 示例

在 `n` 表插入数据后,自动往日志表 `n_log` 里记录一条:

```sql
delimiter //

create trigger trg_n_after_insert
after insert on n
for each row
begin
    insert into n_log(n_id, action) values (new.id, 'insert');
end //

delimiter ;
```

触发器中可以用 `new` 表示新插入或更新后的行,用 `old` 表示更新前或删除的行。

### 3. 查看与删除触发器

```sql
show triggers;
drop trigger if exists trg_n_after_insert;
```

> 触发器的示例以及「查看与删除」一节是整理时补充的内容。
