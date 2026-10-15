---
title: mysqldump常用参数详解
date: 2026-10-12 10:00:00
tags:
  - MySQL
categories:
  - 数据库
description: 按功能分类整理 mysqldump 的常用参数,包括连接、导出范围、导出对象、锁与一致性、输出格式等,并附常用备份与恢复命令。
---

`mysqldump` 是 MySQL 自带的逻辑备份工具,可以把数据库导出为 SQL 文件。本文按功能分类整理它的常用参数,每个参数附带示例,最后给出几条常用的备份与恢复组合命令。

<!-- more -->

> 说明:本文示例均使用 `-uroot -p` 连接数据库,`-p` 后面不写密码,执行时会提示输入,避免密码留在命令行和 shell 历史里。

## 一、连接相关参数

**`--host`, `-h`**:需要导出的主机。

```bash
mysqldump -uroot -p --host=localhost --all-databases
```

**`--port`, `-P`**:连接数据库的端口号。

```bash
mysqldump -uroot -p --host=127.0.0.1 --port=3306 --all-databases
```

**`--user`, `-u`**:指定连接的用户名。

**`--password`, `-p`**:连接数据库的密码。`-p` 后面直接跟密码时不能有空格(如 `-p123456`),不推荐这样写;只写 `-p` 则会交互式提示输入。

**`--socket`, `-S`**:指定连接 MySQL 的 socket 文件位置,默认路径是 `/tmp/mysql.sock`。

```bash
mysqldump -uroot -p --host=localhost --all-databases --socket=/tmp/mysqld.sock
```

## 二、导出范围相关参数

**`--all-databases`, `-A`**:导出全部数据库。

```bash
mysqldump -uroot -p --all-databases
```

**`--databases`, `-B`**:导出指定的几个数据库,参数后面所有的名字都被当作数据库名。

```bash
mysqldump -uroot -p --databases test mysql
```

**`--tables`**:覆盖 `--databases`(`-B`)参数,后面所有的名字都被当作表名。导出指定表时,通常直接写成"库名 表名1 表名2"即可:

```bash
mysqldump -uroot -p test t1 t2
```

**`--ignore-table`**:不导出指定的表。要忽略多个表时,需要重复多次,每次一个表,每个表必须同时写明数据库名和表名,格式为 `--ignore-table=database.table1 --ignore-table=database.table2`。

```bash
mysqldump -uroot -p --host=localhost --all-databases --ignore-table=mysql.user
```

**`--where`, `-w`**:只导出满足给定 WHERE 条件的记录。如果条件里包含空格或 shell 的特殊字符,一定要用引号括起来。

```bash
mysqldump -uroot -p --host=localhost --all-databases --where="user='root'"
```

**`--no-data`, `-d`**:不导出任何数据,只导出表结构。

```bash
mysqldump -uroot -p --host=localhost --all-databases --no-data
```

**`--no-create-info`, `-t`**:只导出数据,不添加 `CREATE TABLE` 语句。

```bash
mysqldump -uroot -p --host=localhost --all-databases --no-create-info
```

**`--no-create-db`, `-n`**:只导出数据,不添加 `CREATE DATABASE` 语句。

```bash
mysqldump -uroot -p --host=localhost --all-databases --no-create-db
```

## 三、导出对象相关参数

**`--routines`, `-R`**:导出存储过程以及自定义函数。

```bash
mysqldump -uroot -p --host=localhost --all-databases --routines
```

**`--events`, `-E`**:导出事件。

```bash
mysqldump -uroot -p --all-databases --events
```

**`--triggers`**:导出触发器。该选项默认启用,用 `--skip-triggers` 禁用。

```bash
mysqldump -uroot -p --host=localhost --all-databases --triggers
```

## 四、锁与数据一致性相关参数

**`--single-transaction`**:InnoDB 表备份时,通常启用该选项来保证备份的一致性。它的工作原理是把本次会话的隔离级别设为 `REPEATABLE READ`(可重复读),确保导出期间不会看到其他会话已提交的数据。

该选项在导出数据之前提交一个 `BEGIN` 语句,`BEGIN` 不会阻塞任何应用程序,且能保证导出时数据库处于一致性状态。它只适用于多版本存储引擎,即**仅 InnoDB**。本选项和 `--lock-tables` 互斥,因为 `LOCK TABLES` 会使任何挂起的事务隐含提交。要导出大表时,应结合使用 `--quick` 选项。

```bash
mysqldump -uroot -p --host=localhost --all-databases --single-transaction
```

**`--lock-tables`, `-l`**:开始导出前,锁定所有表。用 `READ LOCAL` 锁定表,以允许 MyISAM 表并行插入。对于支持事务的表(如 InnoDB),`--single-transaction` 是更好的选择,因为它根本不需要锁表。

注意:导出多个数据库时,`--lock-tables` 是分别为每个数据库锁表,因此不能保证导出文件中的表在数据库之间的逻辑一致性,不同数据库表的导出状态可能完全不同。

```bash
mysqldump -uroot -p --host=localhost --all-databases --lock-tables
```

**`--lock-all-tables`, `-x`**:请求锁定所有数据库中的所有表,以保证数据的一致性。这是一个全局读锁,并且自动关闭 `--single-transaction` 和 `--lock-tables` 选项。

```bash
mysqldump -uroot -p --host=localhost --all-databases --lock-all-tables
```

**`--add-locks`**:在每个表导出之前增加 `LOCK TABLES`,导出之后 `UNLOCK TABLES`。默认打开,用 `--skip-add-locks` 取消。

```bash
mysqldump -uroot -p --all-databases                       # 默认添加 LOCK 语句
mysqldump -uroot -p --all-databases --skip-add-locks      # 取消 LOCK 语句
```

**`--flush-logs`**:开始导出之前刷新日志。

注意:一次导出多个数据库(使用 `--databases` 或 `--all-databases`)时,会逐个数据库刷新日志,只有使用 `--lock-all-tables` 或 `--master-data` 时例外,此时日志只刷新一次,所有表同时被锁定。因此,如果打算同时导出和刷新日志,应该使用 `--lock-all-tables` 或 `--master-data` 配合 `--flush-logs`。

```bash
mysqldump -uroot -p --all-databases --flush-logs
```

**`--flush-privileges`**:在导出 `mysql` 数据库之后,发出一条 `FLUSH PRIVILEGES` 语句。为了正确恢复,导出 `mysql` 数据库以及依赖 `mysql` 数据库数据的任何时候,都应该使用该选项。

```bash
mysqldump -uroot -p --all-databases --flush-privileges
```

## 五、主从复制与日志相关参数

> **版本提示**:`--master-data`、`--dump-slave`、`--delete-master-logs` 在 MySQL 8.0.26 起已改名为 `--source-data`、`--dump-replica`、`--delete-source-logs`,旧名称被弃用,在较新的版本中可能已被移除。请以你使用的版本的 `mysqldump --help` 为准。

**`--master-data`**:把 binlog 的位置和文件名追加到输出文件中。值为 `1` 时输出 `CHANGE MASTER` 命令;值为 `2` 时,`CHANGE MASTER` 命令前会加上注释。

该选项会打开 `--lock-all-tables`,除非同时指定了 `--single-transaction`(这种情况下,全局读锁只在导出开始时获得很短的时间)。该选项会自动关闭 `--lock-tables`。

```bash
mysqldump -uroot -p --host=localhost --all-databases --master-data=1
mysqldump -uroot -p --host=localhost --all-databases --master-data=2
```

**`--dump-slave`**:在**从库**上导出时,把**主库**的 binlog 位置和文件名追加到导出文件中。值为 `1` 时以 `CHANGE MASTER` 命令的形式输出;值为 `2` 时在命令前增加说明信息,默认值为 `0`。

该选项会打开 `--lock-all-tables`,除非指定了 `--single-transaction`,并且会自动关闭 `--lock-tables`。

```bash
mysqldump -uroot -p --all-databases --dump-slave=1
mysqldump -uroot -p --all-databases --dump-slave=2
```

**`--delete-master-logs`**:主库备份后删除日志。这个参数会自动激活 `--master-data`。

```bash
mysqldump -uroot -p --all-databases --delete-master-logs
```

> 这个选项会删除 binlog,如果有从库依赖这些日志,可能导致复制中断,请谨慎使用。

## 六、输出格式相关参数

**`--opt`**:等同于同时开启 `--add-drop-table`、`--add-locks`、`--create-options`、`--quick`、`--extended-insert`、`--lock-tables`、`--set-charset`、`--disable-keys`。该选项默认开启,可以用 `--skip-opt` 禁用。

```bash
mysqldump -uroot -p --host=localhost --all-databases --opt
mysqldump -uroot -p --host=localhost --all-databases --skip-opt
```

**`--add-drop-table`**:在每个表创建之前添加 `DROP TABLE` 语句。默认打开,使用 `--skip-add-drop-table` 取消。

```bash
mysqldump -uroot -p --all-databases                              # 默认添加 drop 语句
mysqldump -uroot -p --all-databases --skip-add-drop-table        # 取消 drop 语句
```

**`--create-options`, `-a`**:在 `CREATE TABLE` 语句中包括所有 MySQL 特性选项。默认打开。

```bash
mysqldump -uroot -p --all-databases
```

**`--extended-insert`, `-e`**:使用具有多个 `VALUES` 列的 `INSERT` 语法,这样导出文件更小,导入速度也更快。默认打开,使用 `--skip-extended-insert` 取消。

```bash
mysqldump -uroot -p --all-databases
mysqldump -uroot -p --all-databases --skip-extended-insert       # 每行数据一条 INSERT
```

**`--quick`, `-q`**:不缓冲查询,直接导出到标准输出,导出大表时能节省内存。默认打开,使用 `--skip-quick` 取消。

```bash
mysqldump -uroot -p --host=localhost --all-databases
mysqldump -uroot -p --host=localhost --all-databases --skip-quick
```

**`--quote-names`, `-Q`**:使用反引号(`` ` ``)引起表名和列名。默认打开,使用 `--skip-quote-names` 取消。

```bash
mysqldump -uroot -p --host=localhost --all-databases
mysqldump -uroot -p --host=localhost --all-databases --skip-quote-names
```

**`--replace`**:使用 `REPLACE INTO` 取代 `INSERT INTO`。

```bash
mysqldump -uroot -p --host=localhost --all-databases --replace
```

**`--set-charset`**:在输出文件中添加 `SET NAMES default_character_set`。默认打开,使用 `--skip-set-charset` 关闭。

```bash
mysqldump -uroot -p --host=localhost --all-databases
mysqldump -uroot -p --host=localhost --all-databases --skip-set-charset
```

**`--no-set-names`, `-N`**:等同于 `--skip-set-charset`。

```bash
mysqldump -uroot -p --host=localhost --all-databases --no-set-names
```

**`--default-character-set`**:设置导出使用的默认字符集。MySQL 5.7 及以前默认为 `utf8`,8.0 起默认为 `utf8mb4`。

```bash
mysqldump -uroot -p --all-databases --default-character-set=latin1
```

**`--order-by-primary`**:如果存在主键,或者第一个唯一键,则对每个表的记录进行排序。在把 MyISAM 表导出并导入到 InnoDB 表时有效,但会使导出工作花费很长时间。

```bash
mysqldump -uroot -p --host=localhost --all-databases --order-by-primary
```

**`--tz-utc`**:在导出文件顶部设置时区 `TIME_ZONE='+00:00'`,以保证在不同时区导出的 `TIMESTAMP` 数据,或者数据被移动到其他时区时的正确性。

```bash
mysqldump -uroot -p --host=localhost --all-databases --tz-utc
```

## 七、输出到文件相关参数

**`--result-file`, `-r`**:直接输出到指定文件中。该选项应该用在使用回车换行对(`\r\n`)换行的系统上(如 DOS、Windows),确保只使用一行。

```bash
mysqldump -uroot -p --host=localhost --all-databases --result-file=/tmp/mysqldump_result_file.txt
```

**`--tab`, `-T`**:为每个表在给定路径下创建以 tab 分隔的文本文件(一个 `.sql` 结构文件和一个 `.txt` 数据文件)。注意:仅适用于 `mysqldump` 和 MySQL 服务器运行在同一台机器上,并且目标目录需要 MySQL 服务有写入权限(受 `secure_file_priv` 限制)。

```bash
mysqldump -uroot -p --host=localhost test test --tab="/home/mysql"
```

**`--lines-terminated-by`**:输出文件的每行用给定字符串分隔。与 `--tab` 选项一起使用,不能用于 `--databases` 和 `--all-databases` 选项。

```bash
mysqldump -uroot -p --host=localhost test test --tab="/tmp/mysql" --lines-terminated-by="##"
```

> 上面两个示例中的 `test test`,第一个是数据库名,第二个是表名。

**`--log-error`**:把警告和错误信息追加到给定文件中。

```bash
mysqldump -uroot -p --host=localhost --all-databases --log-error=/tmp/mysqldump_error_log.err
```

## 八、其他参数

**`--force`**:在导出过程中忽略出现的 SQL 错误。

```bash
mysqldump -uroot -p --all-databases --force
```

**`--verbose`, `-v`**:输出多种平台信息,显示导出的详细过程。

## 九、常用备份与恢复命令

> 本节为整理时补充的内容,不是原笔记里的。

**1. InnoDB 全库备份(推荐组合)**

```bash
mysqldump -uroot -p \
  --single-transaction --quick \
  --routines --triggers --events \
  --all-databases > all_databases.sql
```

**2. 备份单个数据库**

```bash
mysqldump -uroot -p --single-transaction dbname > dbname.sql
```

**3. 只备份表结构**

```bash
mysqldump -uroot -p --no-data dbname > dbname_schema.sql
```

**4. 备份并压缩**

```bash
mysqldump -uroot -p --single-transaction dbname | gzip > dbname.sql.gz
```

**5. 恢复**

```bash
# 恢复普通 SQL 文件
mysql -uroot -p dbname < dbname.sql

# 恢复压缩文件
gunzip < dbname.sql.gz | mysql -uroot -p dbname
```

## 十、使用提示

1. **InnoDB 优先使用 `--single-transaction`**:不锁表、不影响业务,同时保证一致性;MyISAM 表则需要 `--lock-tables` 或 `--lock-all-tables`。
2. **别忘了导出存储过程、函数、事件、触发器**:触发器默认导出,存储过程与事件需要加 `--routines`、`--events`。
3. **大表导出加 `--quick`**:避免把整个表读到内存里(默认已开启)。
4. **备份后做恢复验证**:定期把备份文件恢复到测试库,确认备份可用。
5. **注意字符集**:恢复时数据库、客户端和备份文件的字符集要一致,否则可能出现乱码。
