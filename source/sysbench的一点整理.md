title: sysbench的一点整理
author: cwen
date:  2017-02-09
update:  2017-02-09
tags:
    - sysbench
    - 测试
    - mysql

---

还记得刚到公司的第一个任务就是搞性能测试，断断续续也是搞了很久，sysbench这个工具也是用了有一段时间了，不敢说熟练，但是也是踩了一些坑，在此做些整理，欢迎诸位指正补充...    <!--more-->

sysbench主要包括一下几种方式的测试：

1. cpu性能
2. 磁盘io性能
3. 调度程序性能
4. 内存分配及传输速度
5. POSIX线程性能
6. 数据库性能(OLTP基准测试)

## 先说安装

一般我是直接从github上copy下来，自己编译

```
git clone https://github.com/akopytov/sysbench.git
// 如果使用0.5 版本: git branch 0.5
./autogen.sh
./configure
make
make install
```
详细安装说明参考 [readme](https://github.com/akopytov/sysbench)

## 只说 oltp

sysbench 的 oltp 主要用于评估测试各种不同系统参数下的数据库负载情况。目前sysbench的数据库测试支持 Mysql、PostgreSQL、Oracle (我的任务主要是做 mysql & tidb 的对比测试), 相比 0.4 版本，后续的版本 oltp 测试主要结合了 lua 脚本，不需要修改源码，通过自定义lua脚本就可以实现不同业务类型的测试。

### 参数

sysbench 版本

```
sysbench --version
sysbench 1.0.1-7fe53e4　       　
```
具体参数
```
sysbench --help
Usage:
  sysbench [options]... [testname] [command]

Commands implemented by most tests: prepare run cleanup help

General options:
  --threads=N                     创建测试线程的数量, 默认为 1
  --events=N                      事件最大数量,默认为 0 ,不限制
  --time=N                        最大执行时间，单位是 s ,默认是 0 ,不限制
  --forced-shutdown=STRING        超过max-time强制中断, 默认是 off
  --thread-stack-size=SIZE        每个线程的堆栈大小, 默认是 64k
  --rate=N                        average transactions rate. 0 for unlimited rate [0]
  --report-interval=N             报告中间统计信息间隔, 0 代表禁止, 默认为 0
  --report-checkpoints=[LIST,...] 转储完全统计信息并在指定时间点复位所有计数器。 参数是逗号分隔值的列表，表示从必须执行报告检查点的测试开始所经过的时间（以秒为单位）。 默认情况下，报告检查点处于关闭状态
  --debug[=on|off]                是否显示更多的调试信息, 默认是off
  --validate[=on|off]             在可能情况下执行验证检查, 默认是off。
  --help[=on|off]                 输出 help 信息, 并退出
  --version[=on|off]              输出版本信息, 并退出
  --config-file=FILENAME          配置文件
  --tx-rate=N                     deprecated alias for --rate [0]
  --max-requests=N                deprecated alias for --events [0]
  --max-time=N                    deprecated alias for --time [0]
  --num-threads=N                 deprecated alias for --threads [1]

Pseudo-Random Numbers Generator options:
  --rand-type=STRING 分布的随机数{uniform(均匀分布),Gaussian(高斯分布),special(空间分布)}。默认是special
  --rand-spec-iter=N 产生数的迭代次数。默认是12
  --rand-spec-pct=N  值的百分比被视为’special’ (for special distribution)。默认是1
  --rand-spec-res=N  'special'的百分比值。默认是75
  --rand-seed=N      seed for random number generator. When 0, the current time is used as a RNG seed. [0]
  --rand-pareto-h=N  参数h用于 pareto 分布[0.2]

Log options:
  --verbosity=N verbosity level {5 - debug, 0 - only critical messages} [3]

  --percentile=N       percentile to calculate in latency statistics (1-100). Use the special value of 0 to disable percentile calculations [95]
  --histogram[=on|off] print latency histogram in report [off]

General database options:

  --db-driver=STRING  specifies database driver to use ('help' to get list of available drivers)
  --db-ps-mode=STRING prepared statements usage mode {auto, disable} [auto]
  --db-debug[=on|off] print database-specific debug information [off]
General database options:

  --db-driver=STRING  specifies database driver to use ('help' to get list of available drivers)
  --db-ps-mode=STRING prepared statements usage mode {auto, disable} [auto]
  --db-debug[=on|off] print database-specific debug information [off]


Compiled-in database drivers:
  mysql - MySQL driver

mysql options:
  --mysql-host=[LIST,...]          MySQL server host [localhost]
  --mysql-port=[LIST,...]          MySQL server port [3306]
  --mysql-socket=[LIST,...]        MySQL socket
  --mysql-user=STRING              MySQL user [sbtest]
  --mysql-password=STRING          MySQL password []
  --mysql-db=STRING                MySQL database name [sbtest]
  --mysql-ssl[=on|off]             use SSL connections, if available in the client library [off]
  --mysql-ssl-cipher=STRING        use specific cipher for SSL connections []
  --mysql-compression[=on|off]     use compression, if available in the client library [off]
  --mysql-debug[=on|off]           trace all client library calls [off]
  --mysql-ignore-errors=[LIST,...] list of errors to ignore, or "all" [1213,1020,1205]
  --mysql-dry-run[=on|off]         Dry run, pretend that all MySQL client API calls are successful without executing them [off]

Compiled-in tests:
  fileio - File I/O test
  cpu - CPU performance test
  memory - Memory functions speed test
  threads - Threads subsystem performance test
  mutex - Mutex performance test

See 'sysbench <testname> help' for a list of options for each test.

sysbench ./lua/oltp_read_write.lua help

oltp_read_write.lua options:
  --distinct_ranges=N           Number of SELECT DISTINCT queries per transaction [1]
  --sum_ranges=N                Number of SELECT SUM() queries per transaction [1]
  --skip_trx[=on|off]           Don't start explicit transactions and execute all queries as in the AUTOCOMMIT mode [off]
  --secondary[=on|off]          Use a secondary index in place of the PRIMARY KEY [off]
  --create_secondary[=on|off]   Create a secondary index in addition to the PRIMARY KEY [on]
  --index_updates=N             Number of UPDATE index queries per transaction [1]
  --range_size=N                Range size for range SELECT queries [100]
  --auto_inc[=on|off]           Use AUTO_INCREMENT column as Primary Key (for MySQL), or its alternatives in other DBMS. When disabled, use client-generated IDs [on]
  --delete_inserts=N            Number of DELETE/INSERT combination per transaction [1]
  --tables=N                    Number of tables [1]
  --mysql_storage_engine=STRING Storage engine, if MySQL is used [innodb]
  --non_index_updates=N         Number of UPDATE non-index queries per transaction [1]
  --table_size=N                Number of rows per table [10000]
  --pgsql_variant=STRING        Use this PostgreSQL variant when running with the PostgreSQL driver. The only currently supported variant is 'redshift'. When enabled, create_secondary is automatically disabled, and delete_inserts is set to 0
  --simple_ranges=N             Number of simple range SELECT queries per transaction [1]
  --order_ranges=N              Number of SELECT ORDER BY queries per transaction [1]
  --range_selects[=on|off]      Enable/disable all range SELECT queries [on]
  --point_selects=N             Number of point SELECT queries per transaction [10]

```

> 参数太多实在不想一句句翻译了...

### 开测

sysbench的测试流程

prepare(准备数据) -> run(运行测试) -> cleanup(清理数据)
目前社区提供的lua脚步

```
bulk_insert.lua
oltp_common.lua
oltp_delete.lua
oltp_insert.lua
oltp_point_select.lua
oltp_read_only.lua
oltp_read_write.lua
oltp_update_index.lua
oltp_update_non_index.lua
oltp_write_only.lua
select_random_points.lua
select_random_ranges.lua
```

以 `oltp_read_only.lua` 为例

```
./sysbench ./lua/oltp_read_only.lua \
    --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=000000 \
    --mysql-db=test --tables=10 --table-size=10000 \
    --report-interval=10 \
    --threads=128 --time=120 \
    prepare/run/cleanup

```

结果解读

```
sysbench 1.0.1-7fe53e4 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 128
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 128 tps: 1208.30 qps: 19436.82 (r/w/o: 17007.47/0.00/2429.35) lat (ms,95%): 158.63 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 128 tps: 1264.26 qps: 20233.57 (r/w/o: 17705.05/0.00/2528.52) lat (ms,95%): 153.02 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 128 tps: 1273.70 qps: 20375.11 (r/w/o: 17827.81/0.00/2547.30) lat (ms,95%): 150.29 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 128 tps: 1287.20 qps: 20598.39 (r/w/o: 18023.89/0.00/2574.50) lat (ms,95%): 147.61 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 128 tps: 1281.13 qps: 20491.35 (r/w/o: 17929.10/0.00/2562.26) lat (ms,95%): 150.29 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 128 tps: 1267.04 qps: 20276.48 (r/w/o: 17742.41/0.00/2534.07) lat (ms,95%): 150.29 err/s: 0.00 reconn/s: 0.00
[ 70s ] thds: 128 tps: 1257.80 qps: 20127.47 (r/w/o: 17611.96/0.00/2515.51) lat (ms,95%): 153.02 err/s: 0.00 reconn/s: 0.00
[ 80s ] thds: 128 tps: 1243.93 qps: 19894.40 (r/w/o: 17406.43/0.00/2487.96) lat (ms,95%): 155.80 err/s: 0.00 reconn/s: 0.00
[ 90s ] thds: 128 tps: 1237.60 qps: 19807.67 (r/w/o: 17332.67/0.00/2475.00) lat (ms,95%): 155.80 err/s: 0.00 reconn/s: 0.00
[ 100s ] thds: 128 tps: 1053.60 qps: 16850.29 (r/w/o: 14742.89/0.00/2107.40) lat (ms,95%): 193.38 err/s: 0.00 reconn/s: 0.00
[ 110s ] thds: 128 tps: 1001.41 qps: 16021.33 (r/w/o: 14018.60/0.00/2002.73) lat (ms,95%): 207.82 err/s: 0.00 reconn/s: 0.00
[ 120s ] thds: 128 tps: 859.76 qps: 13750.85 (r/w/o: 12034.73/0.00/1716.12) lat (ms,95%): 235.74 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            1994860    // 总 select 数量
        write:                           0          // 总update、insert、delete语句数量
        other:                           284980     //commit、unlock tables以及其他mutex的数量
        total:                           2279840
    transactions:                        142490 (1186.20 per sec.)   //通常需要关注的数字(TPS)
    queries:                             2279840 (18979.13 per sec.) //通常需要关注的数字(QPS)
    ignored errors:                      0      (0.00 per sec.)  //忽略的错误数
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          120.1216s  //即time指定的压测实际
    total number of events:              142490     //总的事件数，一般与transactions相同

Latency (ms):
         min:                                  8.52
         avg:                                107.84
         max:                                480.08
         95th percentile:                    170.48  //95%的语句的平均响应时间
         sum:                            15365843.76

Threads fairness:
    events (avg/stddev):           1113.2031/14.76
    execution time (avg/stddev):   120.0457/0.04
```

我们一般关注的指标主要有:

* response time avg: 平均响应时间。（后面的95%的大小可以通过--percentile=98的方式去更改）
* transactions: 精确的说是这一项后面的TPS 。但如果使用了-skip-trx=on，这项事务数恒为0，需要用total number of events 去除以总时间，得到tps（其实还可以分为读tps和写tps）
* queries: 用它除以总时间，得到吞吐量QPS
* 当然还有一些系统层面的cpu,io,mem相关指标

### 建议

> sysbench 在最近刚升级了最新版(以上使用的就是最新版),有很多改进之处,让我最惊喜的是prepare数据相比之前变为多线程,准备数据更快, 降低了在prepare数据上耗费的时间(虽然之前自己用别的办法加速prepare数据,但是方法比较挫,这里就不做介绍了)

> 注意新版本相比 1.0 和 0.5 有许多参数的改变
当然你也可以指定自己定义的lua脚步,实现不同业务类型的测试


## 参考

1. [github doc](https://github.com/akopytov/sysbench)
2. [sysbench 0.5使用手册](http://mingxinglai.com/cn/2013/07/sysbench/)
3. [使用sysbench对mysql压力测试](http://seanlook.com/2016/03/28/mysql-sysbench/)
