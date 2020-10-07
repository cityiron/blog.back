---
title: 阿里云RDS采坑记录01
date: 2019-09-22 17:33:11
tags: [otter, canal, mysql, binlog, 阿里云RDS, 原创]
categories: 数据同步
---

[数据同步] 无可奈何之RDS的binlog丢失啦

<!-- more --> 

# 故事原由

上周同事负责的同步服务出现宕机后，由于在忙于另一个重要的项目，线上没有及时处理，后发现同步数据丢失。我趁机了解了下我们的同步逻辑并对这次异常做一个简单的总结。

# 异常描述

线上我们基于otter的msyql数据同步服务出错，出错后会停止数据同步（我们后续的配置没有从中心节点同步到私有云节点），导致了私有云无法正常启动部分服务。

# 发生背景

- 我们分阿里云（中心节点），北京私有云节点，广州私有云节点等，数据会从中心节点同步到私有云节点
- 中心节点使用了阿里云的RDS MySQL数据库，私有云节点采用自己搭建的MySQL
- 采用基于otter的数据同步服务（otter基于canal）
- 我们采用了xxl-job来做定时调度，因为之前认为它只有DML操作，我们的同步服务没有对它的DDL操作做处理
- xxl-job的机器配置的较低，数据量变大之后XXL_JOB_QRTZ_TRIGGER_LOG的查询语句运行较慢，我们给它增加了个索引
- 我们的同步服务异常后，会停止数据同步
- A服务发布，在中心节点加了配置，私有云节点启动失败
- 查询发现同步服务异常，无法通过binlog的偏移量找到记录，导致无法把中心节点加了的配置同步到私有云节点
- 我们是在一天后对这个问题做的处理

> xxl-job的SQL

```sql
SELECT t.id, t.job_group, t.job_id, t.executor_address, t.executor_handler
  , t.executor_param, t.executor_sharding_param, t.executor_fail_retry_count, t.trigger_time, t.trigger_code
  , t.trigger_msg, t.handle_time, t.handle_code, t.handle_msg
FROM XXL_JOB_QRTZ_TRIGGER_LOG t
WHERE t.job_group = ?
  AND t.job_id = ?
  AND t.trigger_time >= ?
  AND t.trigger_time <= ?
ORDER BY id DESC
LIMIT ?, ?
```

> mysql查看binlog

```sql
// 查看binlog文件列表
mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |    107853 |
+------------------+-----------+
1 row in set (0.00 sec)


// 查看binlog状态
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |   107853 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

# 解决对策
因为同步服务是由于binlog的偏移量问题而失败，偏移量是通过zk节点去获取的，我们去MySQL查询了最新的可用的偏移量，设置到了zk的指定节点，让同步服务正常运行。（我们的配置同步在新偏移量之后，所以启动后能够从中心节点同步到私有云节点）

这个是临时的解决方案，我们没法找到之前的binlog的完整记录

# 思考问题

## 为什么过了一天，同步服务会启动失败？

同步服务失败，因为binlog找不到。MySQL我们自己安装的话binlog配置默认是不清理的，但是RDS上不是这样的。
下面是RDS默认配置：
```text
- 保留时长：默认值为18，表示实例空间内默认保存最近18个小时内的Binlog文件，18个小时之前的日志将在备份后（需要开启日志备份）清理。保留时长可选范围值为0~7*24小时。
- 空间使用率不超过：默认值为30%，表示本地Binlog空间使用率大于30%时，系统会从最早的Binlog开始清理，直到空间使用率低于30%。空间使用率不超过可选范围值为0 - 50% 。
- 可用空间保护，默认开启该功能，表示当实例总空间使用率超过80%或实例剩余可用空间不足5GB时，会强制从最早的Binlog开始清理，直到总空间使用率降到80%以下且实例剩余可用空间大于5GB。
```
我们可以看到RDS默认保留时间小于一天，所以我们停了一天后再度开启，导致binlog位置找不到，只能从最新的偏移量同步。这里首先把保留时长调至3天（我们的同步服务不可能停3天，在某些改造项目，同步服务可能会停1-2天），这个需要根据实际场景去设置合理的值。

## binlog不在了，如何补救？

阿里云会把binlog保存到OSS，从OSS下载回来binlog，然后把binlog设置到MySQL指定位置（这块可能是我脑补的）。这块有任何问题可以提工单，毕竟顾客是“上帝”。

本机位置是：
```
/usr/local/mysql/data
```