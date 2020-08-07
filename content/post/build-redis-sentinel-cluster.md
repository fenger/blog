+++
title="Redis哨兵集群搭建"
date=2020-08-07T15:33:42+08:00
draft=true
tags=["redis"]
categories=["Redis"]
url="/2020/08/07/build-redis-sentinel-cluster.html"
toc=true
+++


系统：
```
[root@localhost ~]# cat /etc/redhat-release
CentOS Linux release 7.7.1908 (Core)
```
机器数：3台

# 下载安装redis

## 源码包

redis官网并没有提供二级制包，所以需要下载源码自行构建安装。这里下载的是当前最新的稳定版本`6.0.6`。

## 依赖

redis构建依赖`gcc`，所以需要安装:

```shell
yum install gcc-c++ -y
```

## 构建

解压下载的包，这里是`redis-6.0.6.tar.gz`:

```shell
tar -zxf redis-6.0.6.tar.gz

cd redis-6.0.6

make

cd src

make install PREFIX=$install_path
```

`$install_path`为要安装到的目录。

如果构建出现以下类似错误:

```shell
server.c:2403:11: 错误：‘struct redisServer’没有名为‘assert_line’的成员
     server.assert_line = 0;
           ^
server.c:2404:11: 错误：‘struct redisServer’没有名为‘bug_report_start’的成员
     server.bug_report_start = 0;
           ^
server.c:2405:11: 错误：‘struct redisServer’没有名为‘watchdog_period’的成员
     server.watchdog_period = 0;
```
这是因为gcc版本太低导致的（默认Centos7.7会安装`4.8.x`版本的gcc），需要升级到`9.x`，执行以下命令：

```shell
[root@localhost redis-6.0.6]# yum -y install centos-release-scl 
[root@localhost redis-6.0.6]# yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils
[root@localhost redis-6.0.6]# scl enable devtoolset-9 bash
```
以上方式为临时启用，如果要长期使用`gcc9.x`，需要执行以下命令：
```shell
[root@localhost redis-6.0.6]# echo "source /opt/rh/devtoolset-9/enable" >>/etc/profile
```
再次执行编译即可


待安装完成之后，还需要配置文件，将源码解压包下的`redis.conf`和`sentinel.conf`两个文件拷贝到`$install_path/etc/`下

## redis配置文件

1. 绑定ip
```shell
bind 127.0.0.1
```
如果只有本地连接的话，无需修改，如果需要外部访问，可修改为本机ip或`0.0.0.0`

2. 后台运行
```shell
daemonize no
```
默认情况下，redis并非后台运行，改为`yes`即可后台运行。


## sentinel配置文件

1. sentinel监听端口，默认是26379，按需修改
```shell
port 26379
```

2. 配置是否后台运行，可改为yes
```shell
daemonize yes
```

3. pid，无需修改
```shell
pidfile /var/run/redis-sentinel.pid
```

4. 日志文件地址
```shell
logfile ""
```

5. 设置工作目录，无需修改
```shell
dir /tmp
```

6. 设置sentinel监听master地址，格式为：
```shell
sentinel monitor <master-name> <ip> <redis-port> <quorum>
```
- master-name: 自定义的主节点名称
- ip: 主节点ip
- redis-port: 主节点端口
- quorum: 指定quorum个节点检测到主节点出现问题后故障转移，默认2

示例：
```shell
sentinel monitor mymaster 127.0.0.1 6379 2
```

7. 设置sentinel与master的心跳时间（毫秒），默认30秒
```shell
sentinel down-after-milliseconds <master-name> <milliseconds>
```
示例：
```shell
sentinel down-after-milliseconds mymaster 30000
```

8. 指定在发生failover主备切换时最多可以有多少个slave同时对新的master进行 同步，这个数字越小，完成failover所需的时间就越长，但是如果这个数字越大，就意味着越多的slave因为replication而不可用。可以通过将这个值设为 1 来保证每次只有一个slave 处于不能处理命令请求的状态。
```shell
sentinel parallel-syncs <master-name> <numslaves> 
```
示例：
```shell
sentinel parallel-syncs mymaster 1
```

9. 配置failover-timeout

```shell
sentinel failover-timeout <master-name> <milliseconds>
```
failover-timeout 可以用在以下这些方面： 

    1. 同一个sentinel对同一个master两次failover之间的间隔时间。

    2. 当一个slave从一个错误的master那里同步数据开始计算时间。直到slave被纠正为向正确的master那里同步数据时。

    3. 当想要取消一个正在进行的failover所需要的时间。  

    4. 当进行failover时，配置所有slaves指向新的master所需的最大时间。不过，即使过了这个超时，slaves依然会被正确配置为指向master，但是就不按parallel-syncs所配置的规则来了。

示例：
```shell
sentinel failover-timeout mymaster 180000
```
10. 配置某一事件发生时所需执行的脚本

sentinel的`notification-script`和`reconfig-script`是用来配置当某一事件发生时所需要执行的脚本，可以通过脚本来通知管理员，例如当系统运行不正常时发邮件通知相关人员。对于脚本的运行结果有以下规则：

> 若脚本执行后返回1，那么该脚本稍后将会被再次执行，重复次数目前默认为10

> 若脚本执行后返回2，或者比2更高的一个返回值，脚本将不会重复执行。

> 如果脚本在执行过程中由于收到系统中断信号被终止了，则同返回值为1时的行为相同。

> 一个脚本的最大执行时间为60s，如果超过这个时间，脚本将会被一个SIGKILL信号终止，之后重新执行。

1).通知型脚本:当sentinel有任何警告级别的事件发生时（比如说redis实例的主观失效和客观失效等等），将会去调用这个脚本，这时这个脚本应该通过邮件，SMS等方式去通知系统管理员关于系统不正常运行的信息。调用该脚本时，将传给脚本两个参数，一个是事件的类型，一个是事件的描述。如果sentinel.conf配置文件中配置了这个脚本路径，那么必须保证这个脚本存在于这个路径，并且是可执行的，否则sentinel无法正常启动成功。

```shell
sentinel notification-script <master-name> <script-path> 
配置示例：
sentinel notification-script mymaster /var/redis/notify.sh
```

2). 当一个master由于failover而发生改变时，这个脚本将会被调用，通知相关的客户端关于master地址已经发生改变的信息。以下参数将会在调用脚本时传给脚本:

```shell
sentinel client-reconfig-script <master-name> <script-path> <master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>、
配置示例：
sentinel client-reconfig-script mymaster /var/redis/reconfig.sh
```

目前总是“failover”, 是“leader”或者“observer”中的一个。 参数 from-ip, from-port, to-ip, to-port是用来和旧的master和新的master(即旧的slave)通信的。这个脚本应该是通用的，能被多次调用，不是针对性的。


# 启动redis节点

节点ip：
- 主节点：10.128.5.30
- 从节点1: 10.128.5.31
- 从节点2: 10.128.5.32

1. 启动主节点：
```shell
[root@localhost bin]# ./redis-server ../etc/redis.conf
```
验证主节点状态
```shell
[root@localhost bin]# ./redis-cli -h 127.0.0.1 -p 6379 ping
PONG
```

2. 启动从节点
```shell
[root@localhost bin]# ./redis-server ../etc/redis.conf --slaveof 10.128.5.31 6379
```

`--slaveof`指定主节点ip和端口号

验证从节点状态
```shell
[root@localhost bin]# ./redis-cli -h 127.0.0.1 -p 6379 ping
PONG
```

3. 验证主从关系

- 主节点:
```shell
[root@localhost bin]# ./redis-cli -h 127.0.0.1 -p 6379 info replication
# Replication
role:master
connected_slaves:2
slave0:ip=10.128.5.31,port=6379,state=online,offset=17049263,lag=0
slave1:ip=10.128.5.32,port=6379,state=online,offset=17049263,lag=0
master_replid:8406f1effceaaa13207927a58dea57e85d6507b6
master_replid2:83213c52c0146d3042f787e5c9180b1fc854b8a4
master_repl_offset:17049537
second_repl_offset:44241
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:16000962
repl_backlog_histlen:1048576
```
可以从`role`字段看出是`master`，`connected_slaves`表名有两个连接的从节点, `slave0`和`slave1`显示具体的ip等信息

- 从节点:
```shell
[root@localhost bin]# ./redis-cli -h 127.0.0.1 -p 6379 info replication
# Replication
role:slave
master_host:10.128.5.30
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:17094918
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:8406f1effceaaa13207927a58dea57e85d6507b6
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:17094918
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:16046343
repl_backlog_histlen:1048576
```
可以从`role`字段看出是`slave`, `master_*`指定主节点的信息


# 启动Sentinel节点
redis和sentinel是不同的进程，所以使用的方案是每台redis进程的机器上启动一个sentinel进程
修改`redis_path/etc/sentinel.conf`文件内容（三台机器都需要改），指定主节点redis的ip和端口号:

```shell
sentinel monitor mymaster 10.128.5.30 6379 2
```

分别启动三台sentinel:

```shell
[root@localhost bin]# ./redis-sentinel ../etc/sentinel.conf
```

查看状态:
```shell
[root@localhost bin]# ./redis-cli -h 127.0.0.1 -p 26379 info Sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=10.128.5.30:6379,slaves=2,sentinels=3
```

# 参考
1. https://blog.csdn.net/sunbocong/article/details/85252071
2. https://blog.csdn.net/bktytcc/article/details/107162787



