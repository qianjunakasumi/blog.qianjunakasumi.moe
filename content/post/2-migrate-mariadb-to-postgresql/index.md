---
title: 'Vaultwarden 数据库迁移之 MariaDB 至 PostgreSQL'
description: 'MySQL 日渐式微；PostgreSQL 日渐强盛。有限的资源条件和增长的业务需求，推动着本次迁移的计划和执行。'
slug: vaultwarden-migrate-mariadb-to-postgresql
keywords:
    - migrate
    - mariadb
    - postgresql
    - 迁移
date: 2024-11-04 13:00:00+0900
image: cover.jpg
categories:
    - Infrastructure
tags:
    - Mariadb
    - Postgresql
---

## 动机

起初搭建 Vaultwarden 时，服务器只安装了 MariaDB 数据库。
WordPress 需要 MySQL 家族的数据库，正好 Vaultwarden 也支持 MariaDB，
就顺势放在一起了。

后续博客选型中放弃了 WordPress 方案，转而使用 Hugo 静态部署，数据库就只为 Vaultwarden
服务了。随着其他服务需求的增长（后续可能会有一篇 NetBox 文章），要求使用 PostgreSQL 部署。

考虑到经验上的 MySQL 系列（例如 MariaDB）渐渐不再作为初创项目的第一选择，
PostgreSQL 作为新宠，靠其活跃的社区和插件体系，拥有丰富的周边生态，
所以考虑做出此迁移计划。

当然，生产环境中的数据库在绝大多数情况下都没必要重构，除非您是从 NoSQL 回来（逃

## 调研

在 MariaDB 和 PostgreSQL 中迁移的常见方案有：

- 将所有数据 CSV 导出再导入
- 数据迁移工具实现热迁移、同步等
- ETL (?)
- 其他 DBA 工具

本案用例中，数据总量并不多，实时复制等高大上的工具用不着，没有 SLA 保证，
不是核心业务，可以接受停机操作。

所以大致方案为先停止服务，复制数据，恢复服务。~~（半夜割接）~~

原考虑使用 [pgloader](https://pgloader.io/)，但是在模拟环境中发现了 Blob
数据复制的问题，具体表现为 MariaDB 数据库中的原始值与复制后的 PostgreSQL 中不一致。
具体问题没有细致研究。

在模拟环境中使用比较舒服的是 [DBeaver](https://dbeaver.io/) 工具了，
可视化直接在两个数据库中复制，并且没上述问题，遂采用。

Vaultwarden 中有一个比较有意思的表是 `__diesel_schema_migrations`，
它根据使用的数据库类型不同，表中数据也不相同。
也就是说直接将库表结构全量复制会导致 Vaultwarden 误以为您新创建了个不含任何数据的数据库，
于是会重新执行 SQL 创建表等操作，导致已有表冲突（exists），因此迁移库表结构是行不通的，
只迁移业务数据。

![MariaDB Tables](mariadb-tables.png)

![MariaDB __diesel_schema_migrations Table](maria-vaultwarden___diesel_schema_migrations.png)

## 执行

虽然迁移计划是九月初提出的，但真正执行已经十一月秋天尾巴了（

### 验证当前服务

为确保迁移过程顺畅，事务提交正常处理，保持数据前后一致性，首先需要确保两个数据库均为正常状态。

#### MariaDB

验证 MariaDB 服务正常运行：

```bash
$ systemctl status mariadb.service

● mariadb.service - MariaDB xx.xx.xx database server
     Loaded: loaded (/lib/systemd/system/mariadb.service; enabled; preset: enabled)
     Active: active (running) since Tue 2024-xx-xx 17:26:28 JST; xxx ago
       Docs: man:mariadbd(8)
             https://mariadb.com/kb/en/library/systemd/
   Main PID: xxx (mariadbd)
     Status: "Taking your SQL requests now..."
      Tasks: 20 (limit: 2309)
     Memory: 83.7M
        CPU: 5min 53.787s
     CGroup: /system.slice/mariadb.service
             └─xxx /usr/sbin/mariadbd
```

服务正常运行。
使用 `mysqlcheck` 工具进行自检：

```bash
$ mysqlcheck -u root -p --all-databases

mysql.column_stats                                 OK
mysql.columns_priv                                 OK
mysql.db                                           OK
mysql.event                                        OK
mysql.func                                         OK
mysql.global_priv                                  OK
mysql.gtid_slave_pos                               OK
mysql.help_category                                OK
mysql.help_keyword                                 OK
mysql.help_relation                                OK
mysql.help_topic                                   OK
mysql.index_stats                                  OK
mysql.innodb_index_stats                           OK
mysql.innodb_table_stats                           OK
mysql.plugin                                       OK
mysql.proc                                         OK
mysql.procs_priv                                   OK
mysql.proxies_priv                                 OK
mysql.roles_mapping                                OK
mysql.servers                                      OK
mysql.table_stats                                  OK
mysql.tables_priv                                  OK
mysql.time_zone                                    OK
mysql.time_zone_leap_second                        OK
mysql.time_zone_name                               OK
mysql.time_zone_transition                         OK
mysql.time_zone_transition_type                    OK
mysql.transaction_registry                         OK
sys.sys_config                                     OK
```

验证 MariaDB 服务成功。

#### PostgreSQL

验证 PostgreSQL 服务正常运行：

```bash
$ systemctl status postgresql.service

● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) since Fri 2024-xx-xx 14:35:09 JST; xxx ago
   Main PID: xxx (code=exited, status=0/SUCCESS)
        CPU: 1ms

xxx systemd[1]: Starting postgresql.service - PostgreSQL RDBMS...
xxx systemd[1]: Finished postgresql.service - PostgreSQL RDBMS.

$ systemctl status postgresql@16-main.service 

● postgresql@16-main.service - PostgreSQL Cluster 16-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: active (running) since Fri 2024-xx-xx 14:37:11 JST; xxx ago
    Process: xxx ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 16-main start (code=exited, status=0/SUCCESS)
   Main PID: xxx (postgres)
      Tasks: 8 (limit: 2309)
     Memory: 35.4M
        CPU: 1min 29.026s
     CGroup: /system.slice/system-postgresql.slice/postgresql@16-main.service
             ├─xxx /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
             ├─xxx "postgres: 16/main: checkpointer "
             ├─xxx "postgres: 16/main: background writer "
             ├─xxx "postgres: 16/main: walwriter "
             ├─xxx "postgres: 16/main: autovacuum launcher "
             ├─xxx "postgres: 16/main: logical replication launcher "
             ├─xxx "postgres: 16/main: admin postgres ::1(xxx) idle"
             └─xxx "postgres: 16/main: admin postgres ::1(xxx) idle"

xxx systemd[1]: Starting postgresql@16-main.service - PostgreSQL Cluster 16-main...
xxx systemd[1]: Started postgresql@16-main.service - PostgreSQL Cluster 16-main.
```

笔者这里的 PostgreSQL 是新装的，故无需检查其他部分。所有数据库服务均已准备就绪。

### 中断服务，备份数据库

为了避免数据库迁移过程中数据发生意外更改，造成前后数据不一致或丢失等情况，需要将数据库上层的应用全部停止关闭。
当确保对应服务全部关闭后，使用前期建设中写好了备份数据库的脚本，再执行一遍完成备份。

### 启动 PostgreSQL 的 Vaultwarden 实例

新的 Vaultwarden 实例启动后，连接到 PostgreSQL 数据库后会自动创建好对应的表。
查看 vaultwarden 实例日志：

```bash
[INFO] Using saved config from `data/config.json` for configuration.
[WARNING] The following environment variables are being overridden by the config.json file.
[2024-11-04 06:42:41.737][start][INFO] Rocket has launched from http://0.0.0.0:80
```

确认服务正常运行无误，进入数据库检查

![PostgreSQL Tables](pg-tables.png)

![PostgreSQL __diesel_schema_migrations Table](pg-vaultwarden___diesel_schema_migrations.png)

表结构已经全部就绪。

### 执行数据复制

本次迁移使用 [DBeaver](https://dbeaver.io/) 中的数据传送功能。
选中除 `__diesel_schema_migrations` 外的其他表，导出数据，导出为数据表，进入表映射

![表映射](table-map.png)

![数据加载设置](load-setting.png)

其中需要禁用批处理，迁移过程分为两次，第一次关闭忽略重复行错误，第二次开启忽略重复行错误。
两次迁移的原因是部分数据存在约束，入库时外键尚未准备好。


![确认执行](trans-confirm.png)

继续后，迁移工作开始，等待迁移完成。

### 检查迁移后数据库内容，恢复服务

检查数据库两边的数据信息，确认是否缺少复制、数据不一致等情况。无误即可将服务重新启动。

### 服务验证

进入 Vaultwarden Admin 后台（/admin），检查服务信息

![Vaultwarden Diagnostics](vaultwarden-diag.png)

数据库已经成功更改为 PostgreSQL。
验证 Web 及客户端各个功能，直到全部无误。

访问 /alive ，接口应能正常返回时间信息。

## 结论

大功告成，数据库迁移完毕 😎

## Cover

[ゆめねこ🌸](https://www.pixiv.net/users/28223718)様のイラスト「[楓](https://www.pixiv.net/artworks/123962497)」です。
もしこのイラストが気に入ったら、ぜひ[「ゆめねこ🌸」様のホームページ](https://www.pixiv.net/users/28223718)もチェックしてみてくださいね！

[![楓・ゆめねこ🌸](cover.jpg)](https://www.pixiv.net/artworks/123962497)

If you believe it is being abused, please send an email to qianjunakasumi@outlook.com.
