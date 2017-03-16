title: mysql 配置半同步复制
author: cwen
date:  2017-03-16
update:  2017-03-16
tags:
    - mysql
    - slave
    - binlog
    - 半同步

---

整理一下 mysql 开启半同步复制的操作步骤 <!--more-->

## 配置异步复制

示例 IP
```
12.34.56.789- Master Database
12.23.34.456- Slave Database
```
### 配置 master

修改配置文件
```
sudo vim /etc/mysql/my.conf

// 修改以下配置
server-id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log
binlog_do_db            = newdatabase  // 可选 ; 设置同步的数据库
```
> 当然这些参数你也可以在 mysql 客户端修改 eg: set global server_id=1;

重启 mysql

```
sudo service mysql restart
```

设置给给予从库权限

```
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

读取 master binlog pos

```
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      107 | newdatabase  |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)
```

>  如果同步的数据库有数据, 先从master dump->load进 slave; 这样做的时候 binlog pos 读取 dump 文件上

### 配置 slave

先创建需要同步的 database

```
CREATE DATABASE newdatabase;
EXIT;
```

修改配置文件

```
sudo vim /etc/mysql/my.conf

// 修改以下配置
server-id               = 2    // 区别 master
relay-log               = /var/log/mysql/mysql-relay-bin.log
log_bin                 = /var/log/mysql/mysql-bin.log
binlog_do_db            = newdatabase
```

重启 mysql

```
sudo service mysql restart
```

设置 slave 同步master binlog pos

```
CHANGE MASTER TO MASTER_HOST='12.34.56.789',MASTER_USER='slave_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=  107;
```

开启同步

```
START SLAVE;

// 检查同步是否成功
SHOW SLAVE STATUS\G
```

如果在连接的问题，你可以尝试从开始使用命令跳过它
```
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; SLAVE START;
```

## 开启半同步

master install 半同步插件

```
mysql> install plugin rpl_semi_sync_master soname 'semisync_master.so';
// 查看是否加载插件
mysql> show plugins;
rpl_semi_sync_master   | ACTIVE  | REPLICATION     | semisync_master.so | GPL
```

slave install 半同步插件

```
mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';

// 查看是否加载插件
mysql> show plugins;
rpl_semi_sync_slave    | ACTIVE  | REPLICATION    | semisync_slave.so | GPL
```

启动插件

```
// master
set global rpl_semi_sync_master_enabled = on;

// slave
set global rpl_semi_sync_slave_enabled = on;
stop slave IO_THREAD;
start slave IO_THREAD;
```

检查状态

```
show status like '%Rpl_semi_sync%';
```

解释几个重要的
```
Rpl_semi_sync_master_status	是否启用了半同步
Rpl_semi_sync_master_clients	半同步模式下Slave一共有多少个
Rpl_semi_sync_master_no_tx	往slave发送失败的事务数量
Rpl_semi_sync_master_yes_tx	往slave发送成功的事务数量
```
半同步几个参数设置

```
show variables like '%Rpl%';
```
参数解释

```
rpl_semi_sync_master_timeout	Master等待slave响应的时间，单位是毫秒，默认值是10秒，超过这个时间，slave无响应，环境架构将自动转换为异步复制
rpl_semi_sync_master_trace_level	监控等级，一共4个等级（1,16,32,64），后续补充详细。
rpl_semi_sync_master_wait_no_slave	是否允许master 每个事物提交后都要等待slave的receipt信号。默认为on ，每一个事务都会等待，如果slave当掉后，当slave追赶上master的日志时，可以自动的切换为半同步方式，如果为off,则slave追赶上后，也不会采用半同步的方式复制了，需要手工配置。
rpl_stop_slave_timeout
控制stop slave 的执行时间，在重放一个大的事务的时候,突然执行stop slave ,命令 stop slave会执行很久,这个时候可能产生死锁或阻塞,严重影响性能，mysql 5.6可以通过rpl_stop_slave_timeout参数控制stop slave 的执行时间
```

## 参考

1. [How To Set Up Master Slave Replication in MySQL](https://www.digitalocean.com/community/tutorials/how-to-set-up-master-slave-replication-in-mysql)
2. [Mysql5.6.21半同步](http://www.mamicode.com/info-detail-576734.html)
