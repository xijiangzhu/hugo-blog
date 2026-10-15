---
title: Percona Toolkit 常用命令整理
date: 2026-10-12 10:00:00
tags:
  - MySQL
categories:
  - 数据库
description: 整理 Percona Toolkit 中常用的工具用法:在线改表、主从数据校验与修复、数据归档、慢查询杀死与分析、重复索引检查。
---

Percona Toolkit(简称 `pt` 工具集)是一组用于 MySQL 运维的命令行工具,可以在线修改表结构、校验和修复主从数据、归档数据、分析慢日志等。本文整理其中最常用的几个工具的典型用法。

<!-- more -->

## 零、基础知识

### 1. 安装

```bash
# CentOS / RHEL(需先配置 Percona 的 yum 源)
yum install percona-toolkit

# Ubuntu / Debian
apt install percona-toolkit
```

也可以从 Percona 官网下载安装包或二进制压缩包。

### 2. DSN 连接串

很多工具用 DSN(Data Source Name)来指定数据库、表或服务器,格式是逗号分隔的 `键=值`:

| 键 | 含义 |
| --- | --- |
| `h` | 主机 |
| `P` | 端口 |
| `u` | 用户 |
| `p` | 密码 |
| `D` | 数据库 |
| `t` | 表 |

例如:`h=127.0.0.1,P=3306,u=root,D=test,t=t1`。

### 3. 通用选项

| 选项 | 说明 |
| --- | --- |
| `--ask-pass` | 运行时提示输入密码,避免在命令行明文写密码 |
| `--charset` | 指定连接字符集 |
| `--dry-run` | 试运行,不真正执行修改 |
| `--print` | 打印将要执行的语句 |
| `--execute` | 真正执行 |

> **使用原则**:有 `--dry-run`、`--print` 的工具,**先试运行并检查输出,确认无误后再改用 `--execute`**。生产环境操作前先备份数据。

## 一、pt-online-schema-change(在线修改表结构)

在不长时间锁表的情况下修改表结构。原理是创建一张结构修改后的新表,把数据分批拷贝过去,用触发器同步拷贝期间的增量,最后原子地替换表名。

### 1. 示例命令

```bash
pt-online-schema-change -h 127.0.0.1 -u root -P 3306 D=test,t=t1 \
--ask-pass \
--alter="ADD COLUMN name varchar(16) NOT NULL DEFAULT '' COMMENT '名字'" \
--alter-foreign-keys-method=auto \
--no-drop-old-table \
--charset=utf8 \
--max-load=Threads_running=25 \
--recursion-method=dsn=D=test,t=dsns \
--max-lag=30 \
--no-check-replication-filters \
--dry-run --print
```

确认无误后,把最后一行的 `--dry-run --print` 换成 `--execute` 正式执行。

### 2. 参数说明

| 参数 | 说明 |
| --- | --- |
| `D=test,t=t1` | 要修改的数据库和表 |
| `--alter` | 要执行的修改内容(不需要写 `ALTER TABLE 表名`) |
| `--alter-foreign-keys-method=auto` | 处理外键引用的方式,`auto` 让工具自动选择合适的方法 |
| `--no-drop-old-table` | 改表完成后**不删除**旧表(旧表会被改名保留,方便回退) |
| `--max-load=Threads_running=25` | 当 `Threads_running` 超过 25 时暂停拷贝数据,降低对线上的影响 |
| `--max-lag=30` | 从库延迟超过 30 秒时暂停拷贝数据 |
| `--recursion-method=dsn=D=test,t=dsns` | 从 `test.dsns` 表中读取从库信息,用于检查延迟 |
| `--no-check-replication-filters` | 不检查复制过滤规则 |
| `--dry-run` | 试运行:创建并修改新表结构,但不拷贝数据、不替换原表 |
| `--execute` | 正式执行 |

> `--dry-run` 和 `--execute` 必须二选一。

### 3. dsns 表的创建

`--recursion-method=dsn=...` 指定的表需要提前创建,并写入从库的连接信息:

```sql
CREATE TABLE test.dsns (
  id int(11) NOT NULL AUTO_INCREMENT,
  parent_id int(11) DEFAULT NULL,
  dsn varchar(255) NOT NULL,
  PRIMARY KEY (id)
);

INSERT INTO test.dsns (dsn) VALUES ('h=192.168.1.2,P=3306,u=root');
```

## 二、pt-table-checksum 与 pt-table-sync(主从数据校验与修复)

两个工具配合使用:先用 `pt-table-checksum` 在主库上发现主从数据不一致,再用 `pt-table-sync` 修复。

### 1. 校验数据一致性

在**主库**上执行:

```bash
pt-table-checksum \
--nocheck-replication-filters \
--no-check-binlog-format \
--replicate=test.checksums \
--recursion-method=dsn=D=test,t=dsns \
--databases=test h=127.0.0.1,u=root,P=3306 \
--ask-pass
```

| 参数 | 说明 |
| --- | --- |
| `--nocheck-replication-filters` | 不检查复制过滤规则 |
| `--no-check-binlog-format` | 不检查 binlog 格式(默认要求为 STATEMENT,否则会报错) |
| `--replicate=test.checksums` | 把校验结果保存到 `test.checksums` 表,会通过复制同步到从库 |
| `--recursion-method=dsn=...` | 从 `dsns` 表获取从库列表 |
| `--databases=test` | 只校验 `test` 库 |
| `h=127.0.0.1,u=root,P=3306` | 主库的连接信息 |

结果中的 `DIFFS` 列大于 0,表示该表主从数据存在差异。

### 2. 修复数据不一致

```bash
pt-table-sync \
--replicate=test.checksums \
--sync-to-master h=127.0.0.1,P=3307,u=root \
--ask-pass \
--print
```

| 参数 | 说明 |
| --- | --- |
| `--replicate=test.checksums` | 根据校验结果表,只同步有差异的数据块 |
| `--sync-to-master` | 把后面的 DSN 当作**从库**,让它和自己的主库保持一致 |
| `h=127.0.0.1,P=3307,u=root` | 要修复的从库连接信息 |
| `--print` | 只打印将要执行的修复语句,不真正执行 |

**务必先用 `--print` 检查输出的语句,确认无误后,再换成 `--execute` 执行。**

## 三、pt-archiver(数据归档)

按条件把一张表的数据归档到另一张表或文件中,可选择是否删除源数据,分批执行,对线上影响较小。

> 原笔记标题写的是 `pt-archive`,实际命令是 `pt-archiver`。

```bash
pt-archiver \
--source h=127.0.0.1,P=3306,u=root,D=test,t=t1 \
--dest h=127.0.0.1,P=3306,u=root,D=test,t=t2 \
--ask-pass \
--where "id<5000" \
--charset=utf8 \
--no-check-charset \
--bulk-insert \
--progress 10000 \
--limit 10000 \
--no-delete \
--statistics
```

| 参数 | 说明 |
| --- | --- |
| `--source` | 源表连接信息 |
| `--dest` | 目标表连接信息 |
| `--where "id<5000"` | 归档的条件,这里是 `id` 小于 5000 的数据 |
| `--charset=utf8` | 连接字符集 |
| `--no-check-charset` | 不检查源表和目标表的字符集是否一致 |
| `--bulk-insert` | 批量插入到目标表(使用 `LOAD DATA LOCAL INFILE`),速度更快 |
| `--progress 10000` | 每处理 10000 行输出一次进度 |
| `--limit 10000` | 每次批量取 10000 行 |
| `--no-delete` | **不删除源表数据**,只复制 |
| `--statistics` | 结束时输出统计信息(耗时、行数等) |

> 去掉 `--no-delete`,工具会在归档后**删除源表中已归档的数据**,请确认后再使用。只想删除数据而不归档,可使用 `--purge`。

## 四、pt-kill(杀死符合条件的查询)

按条件自动杀死长时间运行的查询或连接。

```bash
pt-kill \
--host=127.0.0.1 \
--user=root \
--port=3306 \
--ask-pass \
--match-info "SELECT|select" \
--busy-time=100 \
--victim all \
--interval 5 \
--kill \
--print
```

| 参数 | 说明 |
| --- | --- |
| `--match-info "SELECT\|select"` | 匹配查询语句内容符合该正则的连接(这里是含 SELECT 的语句) |
| `--busy-time=100` | 只匹配已经运行了 100 秒以上的查询 |
| `--victim all` | 匹配到的**所有**查询都杀死(默认只杀最久的一个) |
| `--interval 5` | 每 5 秒检查一次 |
| `--kill` | 真正杀死匹配到的连接 |
| `--print` | 打印被杀死的查询 |

> 这个命令会真正杀死连接,先去掉 `--kill`、只保留 `--print` 观察会匹配到哪些语句,确认无误后再加上 `--kill`。持续运行时可以配合 `--daemonize` 放到后台。

## 五、pt-query-digest(慢查询日志分析)

分析慢查询日志,汇总出最耗时、最频繁的 SQL。原笔记只记录了按时间范围过滤的两个参数:

```bash
pt-query-digest \
--since "yyyy-mm-dd hh:mm:ss" \
--until "yyyy-mm-dd hh:mm:ss" \
/var/lib/mysql/slow.log
```

| 参数 | 说明 |
| --- | --- |
| `--since` | 只分析该时间之后的记录 |
| `--until` | 只分析该时间之前的记录 |

> 命令末尾要指定慢查询日志文件(路径按实际环境替换,可用 `show variables like 'slow_query_log_file';` 查看)。结果较长时,可以重定向保存:`... slow.log > report.txt`。

## 六、pt-duplicate-key-checker(重复索引检查)

检查表中重复或冗余的索引和外键。

```bash
pt-duplicate-key-checker \
--host=127.0.0.1 \
--user=root \
--port=3306 \
--ask-pass \
--databases=test \
--tables=t1
```

| 参数 | 说明 |
| --- | --- |
| `--databases=test` | 指定检查的数据库 |
| `--tables=t1` | 指定检查的表(不写则检查整个库) |

输出结果会列出重复的索引,并给出对应的 `ALTER TABLE ... DROP INDEX ...` 建议语句。

## 七、使用提示

1. **先试运行再执行**:`--dry-run`、`--print` 看过没问题,再用 `--execute`、`--kill` 等真正生效的参数。
2. **密码不要写在命令行**:使用 `--ask-pass` 交互输入,避免密码留在 shell 历史里。
3. **选项在命令中可以任意换行**:本文用 `\` 把长命令分行书写,实际执行时也可以写成一行。
4. **注意版本**:不同版本的 Percona Toolkit 和 MySQL 版本之间,个别参数行为有差异,以 `pt-xxx --help` 的输出为准。
5. **控制对线上的影响**:改表、归档时,配合 `--max-load`、`--max-lag`、`--limit` 等参数,避免拖慢业务。
