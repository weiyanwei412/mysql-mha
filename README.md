高可用环境准备
准备两台centos6.9系统的虚拟机
IP地址规划为
192.168.56.200 node1 master
192.168.56.201 node2 slave
192.168.56.1111 vip


[mysqld]

####symi replication settings##
plugin_dir=/usr/local/mysql/lib/plugin
plugin_load = "rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"
rpl_semi_sync_master_enabled=1
rpl_semi_sync_master_timeout=10000 #10s
loose_rpl_semi_sync_slave_enabled = 1

关闭防火墙
service iptables stop
chkconfig --del iptables

设置hosts
cat >/etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.56.200 node1
192.168.56.201 node2
EOF

在192.168.56.200上执行
ssh-keygen -t rsa
 
cd ~/.ssh 
cat id_rsa.pub >>authorized_keys
scp ./* root@node2:/root/.ssh/
ssh node2
ssh node1

创建同步账号
grant replication slave on *.* to 'repl'@'192.168.56.%' identified by 'lrepl';
flush privileges;

在node2上
执行change master语句
change master to master_host='192.168.56.200',master_user='rep',master_password='repl',master_auto_position=1;
start slave;
show slave status\G
设置从库只读
mysql -uroot -pletian318 -e "set global read_only=1"
====================================================================================
创建监控用户（在master上执行，也就是mha120 192.168.56.120）：
grant all privileges on *.* to 'monitor'@'192.168.56.%' identified  by 'letian318'; 
flush  privileges;


在node1  node2 上执行
配置好base 和epel源

yum localinstall -y  mha4mysql-node-0.57-0.el7.noarch.rpm

在node2上执行
yum localinstall -y mha4mysql-manager-0.57-0.el7.noarch.rpm
mkdir -p  /var/log/masterha/app1

运行
解压在/etc下masterha.zip
bash  check_ssh.sh
[root@node2 masterha]# bash check_ssh.sh 
Mon Nov 27 20:42:33 2017 - [info] Reading default configuration from /etc/masterha/masterha_default.conf..
Mon Nov 27 20:42:33 2017 - [info] Reading application default configuration from /etc/masterha/app1.conf..
Mon Nov 27 20:42:33 2017 - [info] Reading server configuration from /etc/masterha/app1.conf..
Mon Nov 27 20:42:33 2017 - [info] Starting SSH connection tests..
Mon Nov 27 20:42:34 2017 - [debug] 
Mon Nov 27 20:42:33 2017 - [debug]  Connecting via SSH from root@node1(192.168.56.200:22) to root@node2(192.168.56.201:22)..
Mon Nov 27 20:42:33 2017 - [debug]   ok.
Mon Nov 27 20:42:34 2017 - [debug] 
Mon Nov 27 20:42:34 2017 - [debug]  Connecting via SSH from root@node2(192.168.56.201:22) to root@node1(192.168.56.200:22)..
Mon Nov 27 20:42:34 2017 - [debug]   ok.
Mon Nov 27 20:42:34 2017 - [info] All SSH connection tests passed successfully.
[root@node2 masterha]# 

[root@node1 masterha]# bash check_repl.sh 
Mon Nov 27 20:42:50 2017 - [info] Reading default configuration from /etc/masterha/masterha_default.conf..
Mon Nov 27 20:42:50 2017 - [info] Reading application default configuration from /etc/masterha/app1.conf..
Mon Nov 27 20:42:50 2017 - [info] Reading server configuration from /etc/masterha/app1.conf..
Mon Nov 27 20:42:50 2017 - [info] MHA::MasterMonitor version 0.57.
Mon Nov 27 20:42:50 2017 - [info] GTID failover mode = 1
Mon Nov 27 20:42:50 2017 - [info] Dead Servers:
Mon Nov 27 20:42:50 2017 - [info] Alive Servers:
Mon Nov 27 20:42:50 2017 - [info]   node1(192.168.56.200:3306)
Mon Nov 27 20:42:50 2017 - [info]   node2(192.168.56.201:3306)
Mon Nov 27 20:42:50 2017 - [info] Alive Slaves:
Mon Nov 27 20:42:50 2017 - [info]   node2(192.168.56.201:3306)  Version=5.7.20-log (oldest major version between slaves) log-bin:enabled
Mon Nov 27 20:42:50 2017 - [info]     GTID ON
Mon Nov 27 20:42:50 2017 - [info]     Replicating from 192.168.56.200(192.168.56.200:3306)
Mon Nov 27 20:42:50 2017 - [info]     Primary candidate for the new Master (candidate_master is set)
Mon Nov 27 20:42:50 2017 - [info] Current Alive Master: node1(192.168.56.200:3306)
Mon Nov 27 20:42:50 2017 - [info] Checking slave configurations..
Mon Nov 27 20:42:50 2017 - [info]  read_only=1 is not set on slave node2(192.168.56.201:3306).
Mon Nov 27 20:42:50 2017 - [info] Checking replication filtering settings..
Mon Nov 27 20:42:50 2017 - [info]  binlog_do_db= , binlog_ignore_db= 
Mon Nov 27 20:42:50 2017 - [info]  Replication filtering check ok.
Mon Nov 27 20:42:50 2017 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Mon Nov 27 20:42:50 2017 - [info] Checking SSH publickey authentication settings on the current master..
Mon Nov 27 20:42:50 2017 - [info] HealthCheck: SSH to node1 is reachable.
Mon Nov 27 20:42:50 2017 - [info] 
node1(192.168.56.200:3306) (current master)
 +--node2(192.168.56.201:3306)

Mon Nov 27 20:42:50 2017 - [info] Checking replication health on node2..
Mon Nov 27 20:42:50 2017 - [info]  ok.
Mon Nov 27 20:42:50 2017 - [info] Checking master_ip_failover_script status:
Mon Nov 27 20:42:50 2017 - [info]   /etc/masterha/master_ip_failover --command=status --ssh_user=root --orig_master_host=node1 --orig_master_ip=192.168.56.200 --orig_master_port=3306 


IN SCRIPT TEST====/sbin/ifconfig eth0:1 down==/sbin/ifconfig eth0:1 192.168.56.111/24===

Checking the Status of the script.. OK 
Mon Nov 27 20:42:50 2017 - [info]  OK.
Mon Nov 27 20:42:50 2017 - [warning] shutdown_script is not defined.
Mon Nov 27 20:42:50 2017 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
[root@node2 masterha]# 
发现主库挂上了我们定义好的vip 
[root@node1 masterha]# ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:6f:b6:53 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.200/24 brd 192.168.56.255 scope global eth0
    inet 192.168.56.111/24 brd 192.168.56.255 scope global secondary eth0:1
    inet6 fe80::20c:29ff:fe6f:b653/64 scope link 
       valid_lft forever preferred_lft forever
[root@node1 masterha]# 

运行管理监控脚本
bash    start-manager.sh


模拟主库挂掉
在node1上
/etc/init.d/mysqld stop

查看mannager  日志
Mon Nov 27 18:59:15 2017 - [info] Ping(SELECT) succeeded, waiting until MySQL doesn't respond..
Mon Nov 27 19:00:24 2017 - [warning] Got error on MySQL select ping: 2006 (MySQL server has gone away)
Mon Nov 27 19:00:24 2017 - [info] Executing SSH check script: exit 0
Mon Nov 27 19:00:24 2017 - [info] HealthCheck: SSH to node1 is reachable.
Mon Nov 27 19:00:25 2017 - [warning] Got error on MySQL connect: 2013 (Lost connection to MySQL server at 'reading initial communication packet', system error: 111)
Mon Nov 27 19:00:25 2017 - [warning] Connection failed 2 time(s)..
Mon Nov 27 19:00:26 2017 - [warning] Got error on MySQL connect: 2013 (Lost connection to MySQL server at 'reading initial communication packet', system error: 111)
Mon Nov 27 19:00:26 2017 - [warning] Connection failed 3 time(s)..
Mon Nov 27 19:00:27 2017 - [warning] Got error on MySQL connect: 2013 (Lost connection to MySQL server at 'reading initial communication packet', system error: 111)
Mon Nov 27 19:00:27 2017 - [warning] Connection failed 4 time(s)..
Mon Nov 27 19:00:27 2017 - [warning] Master is not reachable from health checker!
Mon Nov 27 19:00:27 2017 - [warning] Master node1(192.168.56.200:3306) is not reachable!
Mon Nov 27 19:00:27 2017 - [warning] SSH is reachable.
Mon Nov 27 19:00:27 2017 - [info] Connecting to a master server failed. Reading configuration file /etc/masterha/masterha_default.conf and /etc/masterha/app1.conf again, and trying to connect to all servers to check server status..
Mon Nov 27 19:00:27 2017 - [info] Reading default configuration from /etc/masterha/masterha_default.conf..
Mon Nov 27 19:00:27 2017 - [info] Reading application default configuration from /etc/masterha/app1.conf..
Mon Nov 27 19:00:27 2017 - [info] Reading server configuration from /etc/masterha/app1.conf..
Mon Nov 27 19:00:27 2017 - [info] GTID failover mode = 1
Mon Nov 27 19:00:27 2017 - [info] Dead Servers:
Mon Nov 27 19:00:27 2017 - [info]   node1(192.168.56.200:3306)
Mon Nov 27 19:00:27 2017 - [info] Alive Servers:
Mon Nov 27 19:00:27 2017 - [info]   node2(192.168.56.201:3306)
Mon Nov 27 19:00:27 2017 - [info] Alive Slaves:
Mon Nov 27 19:00:27 2017 - [info]   node2(192.168.56.201:3306)  Version=5.7.20-log (oldest major version between slaves) log-bin:enabled
Mon Nov 27 19:00:27 2017 - [info]     GTID ON
Mon Nov 27 19:00:27 2017 - [info]     Replicating from 192.168.56.200(192.168.56.200:3306)
Mon Nov 27 19:00:27 2017 - [info]     Primary candidate for the new Master (candidate_master is set)
Mon Nov 27 19:00:27 2017 - [info] Checking slave configurations..
Mon Nov 27 19:00:27 2017 - [info]  read_only=1 is not set on slave node2(192.168.56.201:3306).
Mon Nov 27 19:00:27 2017 - [info] Checking replication filtering settings..
Mon Nov 27 19:00:27 2017 - [info]  Replication filtering check ok.
Mon Nov 27 19:00:27 2017 - [info] Master is down!
Mon Nov 27 19:00:27 2017 - [info] Terminating monitoring script.
Mon Nov 27 19:00:27 2017 - [info] Got exit code 20 (Master dead).
Mon Nov 27 19:00:27 2017 - [info] MHA::MasterFailover version 0.57.
Mon Nov 27 19:00:27 2017 - [info] Starting master failover.
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] * Phase 1: Configuration Check Phase..
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] GTID failover mode = 1
Mon Nov 27 19:00:27 2017 - [info] Dead Servers:
Mon Nov 27 19:00:27 2017 - [info]   node1(192.168.56.200:3306)
Mon Nov 27 19:00:27 2017 - [info] Checking master reachability via MySQL(double check)...
Mon Nov 27 19:00:27 2017 - [info]  ok.
Mon Nov 27 19:00:27 2017 - [info] Alive Servers:
Mon Nov 27 19:00:27 2017 - [info]   node2(192.168.56.201:3306)
Mon Nov 27 19:00:27 2017 - [info] Alive Slaves:
Mon Nov 27 19:00:27 2017 - [info]   node2(192.168.56.201:3306)  Version=5.7.20-log (oldest major version between slaves) log-bin:enabled
Mon Nov 27 19:00:27 2017 - [info]     GTID ON
Mon Nov 27 19:00:27 2017 - [info]     Replicating from 192.168.56.200(192.168.56.200:3306)
Mon Nov 27 19:00:27 2017 - [info]     Primary candidate for the new Master (candidate_master is set)
Mon Nov 27 19:00:27 2017 - [info] Starting GTID based failover.
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] ** Phase 1: Configuration Check Phase completed.
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] * Phase 2: Dead Master Shutdown Phase..
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] Forcing shutdown so that applications never connect to the current master..
Mon Nov 27 19:00:27 2017 - [info] Executing master IP deactivation script:
Mon Nov 27 19:00:27 2017 - [info]   /etc/masterha/master_ip_failover --orig_master_host=node1 --orig_master_ip=192.168.56.200 --orig_master_port=3306 --command=stopssh --ssh_user=root  


IN SCRIPT TEST====/sbin/ifconfig eth0:1 down==/sbin/ifconfig eth0:1 192.168.56.111/24===

Disabling the VIP on old master: node1 
Mon Nov 27 19:00:27 2017 - [info]  done.
Mon Nov 27 19:00:27 2017 - [warning] shutdown_script is not set. Skipping explicit shutting down of the dead master.
Mon Nov 27 19:00:27 2017 - [info] * Phase 2: Dead Master Shutdown Phase completed.
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] * Phase 3: Master Recovery Phase..
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] * Phase 3.1: Getting Latest Slaves Phase..
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] The latest binary log file/position on all slaves is mysql-bin.000009:194
Mon Nov 27 19:00:27 2017 - [info] Latest slaves (Slaves that received relay log files to the latest):
Mon Nov 27 19:00:27 2017 - [info]   node2(192.168.56.201:3306)  Version=5.7.20-log (oldest major version between slaves) log-bin:enabled
Mon Nov 27 19:00:27 2017 - [info]     GTID ON
Mon Nov 27 19:00:27 2017 - [info]     Replicating from 192.168.56.200(192.168.56.200:3306)
Mon Nov 27 19:00:27 2017 - [info]     Primary candidate for the new Master (candidate_master is set)
Mon Nov 27 19:00:27 2017 - [info] The oldest binary log file/position on all slaves is mysql-bin.000009:194
Mon Nov 27 19:00:27 2017 - [info] Oldest slaves:
Mon Nov 27 19:00:27 2017 - [info]   node2(192.168.56.201:3306)  Version=5.7.20-log (oldest major version between slaves) log-bin:enabled
Mon Nov 27 19:00:27 2017 - [info]     GTID ON
Mon Nov 27 19:00:27 2017 - [info]     Replicating from 192.168.56.200(192.168.56.200:3306)
Mon Nov 27 19:00:27 2017 - [info]     Primary candidate for the new Master (candidate_master is set)
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] * Phase 3.3: Determining New Master Phase..
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] Searching new master from slaves..
Mon Nov 27 19:00:27 2017 - [info]  Candidate masters from the configuration file:
Mon Nov 27 19:00:27 2017 - [info]   node2(192.168.56.201:3306)  Version=5.7.20-log (oldest major version between slaves) log-bin:enabled
Mon Nov 27 19:00:27 2017 - [info]     GTID ON
Mon Nov 27 19:00:27 2017 - [info]     Replicating from 192.168.56.200(192.168.56.200:3306)
Mon Nov 27 19:00:27 2017 - [info]     Primary candidate for the new Master (candidate_master is set)
Mon Nov 27 19:00:27 2017 - [info]  Non-candidate masters:
Mon Nov 27 19:00:27 2017 - [info]  Searching from candidate_master slaves which have received the latest relay log events..
Mon Nov 27 19:00:27 2017 - [info] New master is node2(192.168.56.201:3306)
Mon Nov 27 19:00:27 2017 - [info] Starting master failover..
Mon Nov 27 19:00:27 2017 - [info] 
From:
node1(192.168.56.200:3306) (current master)
 +--node2(192.168.56.201:3306)

To:
node2(192.168.56.201:3306) (new master)
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] * Phase 3.3: New Master Recovery Phase..
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info]  Waiting all logs to be applied.. 
Mon Nov 27 19:00:27 2017 - [info]   done.
Mon Nov 27 19:00:27 2017 - [info] Getting new master's binlog name and position..
Mon Nov 27 19:00:27 2017 - [info]  mysql-bin.000003:234
Mon Nov 27 19:00:27 2017 - [info]  All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='node2 or 192.168.56.201', MASTER_PORT=3306, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='xxx';
Mon Nov 27 19:00:27 2017 - [info] Master Recovery succeeded. File:Pos:Exec_Gtid_Set: mysql-bin.000003, 234, c9a69d41-d3ba-11e7-94ff-000c296fb653:1-9,
f33d56b8-d3ba-11e7-8643-000c292e5673:1-7
Mon Nov 27 19:00:27 2017 - [info] Executing master IP activate script:
Mon Nov 27 19:00:27 2017 - [info]   /etc/masterha/master_ip_failover --command=start --ssh_user=root --orig_master_host=node1 --orig_master_ip=192.168.56.200 --orig_master_port=3306 --new_master_host=node2 --new_master_ip=192.168.56.201 --new_master_port=3306 --new_master_user='monitor'   --new_master_password=xxx
Unknown option: new_master_user
Unknown option: new_master_password


IN SCRIPT TEST====/sbin/ifconfig eth0:1 down==/sbin/ifconfig eth0:1 192.168.56.111/24===

Enabling the VIP - 192.168.56.111/24 on the new master - node2 
Mon Nov 27 19:00:27 2017 - [info]  OK.
Mon Nov 27 19:00:27 2017 - [info] ** Finished master recovery successfully.
Mon Nov 27 19:00:27 2017 - [info] * Phase 3: Master Recovery Phase completed.
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] * Phase 4: Slaves Recovery Phase..
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] * Phase 4.1: Starting Slaves in parallel..
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] All new slave servers recovered successfully.
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] * Phase 5: New master cleanup phase..
Mon Nov 27 19:00:27 2017 - [info] 
Mon Nov 27 19:00:27 2017 - [info] Resetting slave info on the new master..
Mon Nov 27 19:00:27 2017 - [info]  node2: Resetting slave info succeeded.
Mon Nov 27 19:00:27 2017 - [info] Master failover to node2(192.168.56.201:3306) completed successfully.
Mon Nov 27 19:00:27 2017 - [info] 

----- Failover Report -----

app1: MySQL Master failover node1(192.168.56.200:3306) to node2(192.168.56.201:3306) succeeded

Master node1(192.168.56.200:3306) is down!

Check MHA Manager logs at node1:/var/log/masterha/app1/app1.log for details.

Started automated(non-interactive) failover.
Invalidated master IP address on node1(192.168.56.200:3306)
Selected node2(192.168.56.201:3306) as a new master.
node2(192.168.56.201:3306): OK: Applying all logs succeeded.
node2(192.168.56.201:3306): OK: Activated master IP address.
node2(192.168.56.201:3306): Resetting slave info succeeded.
Master failover to node2(192.168.56.201:3306) completed successfully.


发现vip 已经挂在了node2节点上，

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:2e:56:73 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.201/24 brd 192.168.56.255 scope global eth0
    inet 192.168.56.111/24 brd 192.168.56.255 scope global secondary eth0:1
    inet6 fe80::20c:29ff:fe6f:b653/64 scope link 
       valid_lft forever preferred_lft forever

===============================================================
配置文件
===============================================================
[root@node1 masterha]# cat app1.conf 
[server default]
#mha manager工作目录
manager_workdir = /var/log/masterha/app1
manager_log = /var/log/masterha/app1/app1.log
remote_workdir = /var/log/masterha/app1

[server1]
hostname=node1
master_binlog_dir = /data/mysql/mysql3306/logs
candidate_master = 1
check_repl_delay = 0     #用防止master故障时，切换时slave有延迟，卡在那里切不过来。

[server2]
hostname=node2
master_binlog_dir=/data/mysql/mysql3306/logs
candidate_master=1
check_repl_delay=0
==========================================================
[root@node1 masterha]# cat masterha_default.conf 
[server default]
#MySQL的用户和密码
user=monitor
password=letian318

#系统ssh用户
ssh_user=root

#复制用户
repl_user=repl
repl_password= repl


#监控
ping_interval=1
#shutdown_script=""

#切换调用的脚本
master_ip_failover_script= /etc/masterha/master_ip_failover
master_ip_online_change_script= /etc/masterha/master_ip_online_change
=============================================================
cat master_ip_failover
#!/usr/bin/env perl

use strict;
use warnings FATAL => 'all';

use Getopt::Long;

my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);

my $vip = '192.168.56.111/24';  # Virtual IP 
my $key = "1"; 
my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";

GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);

exit &main();

sub main {

    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n"; 

    if ( $command eq "stop" || $command eq "stopssh" ) {

        # $orig_master_host, $orig_master_ip, $orig_master_port are passed.
        # If you manage master ip address at global catalog database,
        # invalidate orig_master_ip here.
        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {

        # all arguments are passed.
        # If you manage master ip address at global catalog database,
        # activate new_master_ip here.
        # You can also grant write access (create user, set read_only=0, etc) here.
        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n"; 
        `ssh $ssh_user\@$orig_master_host \" $ssh_start_vip \"`;
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}

# A simple system call that enable the VIP on the new master 
sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
# A simple system call that disable the VIP on the old_master
sub stop_vip() {
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}

sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}

[root@node1 masterha]# cat master_ip_online_change 
#!/usr/bin/env perl

#  Copyright (C) 2011 DeNA Co.,Ltd.
#
#  This program is free software; you can redistribute it and/or modify
#  it under the terms of the GNU General Public License as published by
#  the Free Software Foundation; either version 2 of the License, or
#  (at your option) any later version.
#
#  This program is distributed in the hope that it will be useful,
#  but WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#  GNU General Public License for more details.
#
#  You should have received a copy of the GNU General Public License
#   along with this program; if not, write to the Free Software
#  Foundation, Inc.,
#  51 Franklin Street, Fifth Floor, Boston, MA  02110-1301  USA

## Note: This is a sample script and is not complete. Modify the script based on your environment.

use strict;
use warnings FATAL => 'all';

use Getopt::Long;
use MHA::DBHelper;
use MHA::NodeUtil;
use Time::HiRes qw( sleep gettimeofday tv_interval );
use Data::Dumper;

my $_tstart;
my $_running_interval = 0.1;
#添加vip定义
my $vip = "192.168.56.111";
my $if = "eth0";

my (
  $command,              $orig_master_is_new_slave, $orig_master_host,
  $orig_master_ip,       $orig_master_port,         $orig_master_user,
  $orig_master_password, $orig_master_ssh_user,     $new_master_host,
  $new_master_ip,        $new_master_port,          $new_master_user,
  $new_master_password,  $new_master_ssh_user,
);
GetOptions(
  'command=s'                => \$command,
  'orig_master_is_new_slave' => \$orig_master_is_new_slave,
  'orig_master_host=s'       => \$orig_master_host,
  'orig_master_ip=s'         => \$orig_master_ip,
  'orig_master_port=i'       => \$orig_master_port,
  'orig_master_user=s'       => \$orig_master_user,
  'orig_master_password=s'   => \$orig_master_password,
  'orig_master_ssh_user=s'   => \$orig_master_ssh_user,
  'new_master_host=s'        => \$new_master_host,
  'new_master_ip=s'          => \$new_master_ip,
  'new_master_port=i'        => \$new_master_port,
  'new_master_user=s'        => \$new_master_user,
  'new_master_password=s'    => \$new_master_password,
  'new_master_ssh_user=s'    => \$new_master_ssh_user,
);

exit &main();
sub drop_vip {
        my $output = `ssh -o ConnectTimeout=15  -o ConnectionAttempts=3 $orig_master_host /sbin/ip addr del $vip/32 dev $if`;
	#mysql里的连接全部干掉
	#FIXME
}
sub add_vip {
        my $output = `ssh -o ConnectTimeout=15  -o ConnectionAttempts=3 $new_master_host /sbin/ip addr add $vip/32 dev $if`;

}


sub current_time_us {
  my ( $sec, $microsec ) = gettimeofday();
  my $curdate = localtime($sec);
  return $curdate . " " . sprintf( "%06d", $microsec );
}

sub sleep_until {
  my $elapsed = tv_interval($_tstart);
  if ( $_running_interval > $elapsed ) {
    sleep( $_running_interval - $elapsed );
  }
}

sub get_threads_util {
  my $dbh                    = shift;
  my $my_connection_id       = shift;
  my $running_time_threshold = shift;
  my $type                   = shift;
  $running_time_threshold = 0 unless ($running_time_threshold);
  $type                   = 0 unless ($type);
  my @threads;

  my $sth = $dbh->prepare("SHOW PROCESSLIST");
  $sth->execute();

  while ( my $ref = $sth->fetchrow_hashref() ) {
    my $id         = $ref->{Id};
    my $user       = $ref->{User};
    my $host       = $ref->{Host};
    my $command    = $ref->{Command};
    my $state      = $ref->{State};
    my $query_time = $ref->{Time};
    my $info       = $ref->{Info};
    $info =~ s/^\s*(.*?)\s*$/$1/ if defined($info);
    next if ( $my_connection_id == $id );
    next if ( defined($query_time) && $query_time < $running_time_threshold );
    next if ( defined($command)    && $command eq "Binlog Dump" );
    next if ( defined($user)       && $user eq "system user" );
    next
      if ( defined($command)
      && $command eq "Sleep"
      && defined($query_time)
      && $query_time >= 1 );

    if ( $type >= 1 ) {
      next if ( defined($command) && $command eq "Sleep" );
      next if ( defined($command) && $command eq "Connect" );
    }

    if ( $type >= 2 ) {
      next if ( defined($info) && $info =~ m/^select/i );
      next if ( defined($info) && $info =~ m/^show/i );
    }

    push @threads, $ref;
  }
  return @threads;
}

sub main {
  if ( $command eq "stop" ) {
    ## Gracefully killing connections on the current master
    # 1. Set read_only= 1 on the new master
    # 2. DROP USER so that no app user can establish new connections
    # 3. Set read_only= 1 on the current master
    # 4. Kill current queries
    # * Any database access failure will result in script die.
    my $exit_code = 1;
    eval {
      ## Setting read_only=1 on the new master (to avoid accident)
      my $new_master_handler = new MHA::DBHelper();

      # args: hostname, port, user, password, raise_error(die_on_error)_or_not
      $new_master_handler->connect( $new_master_ip, $new_master_port,
        $new_master_user, $new_master_password, 1 );
      print current_time_us() . " Set read_only on the new master.. ";
      $new_master_handler->enable_read_only();
      if ( $new_master_handler->is_read_only() ) {
        print "ok.\n";
      }
      else {
        die "Failed!\n";
      }
      $new_master_handler->disconnect();

      # Connecting to the orig master, die if any database error happens
      my $orig_master_handler = new MHA::DBHelper();
      $orig_master_handler->connect( $orig_master_ip, $orig_master_port,
        $orig_master_user, $orig_master_password, 1 );

      ## Drop application user so that nobody can connect. Disabling per-session binlog beforehand
      $orig_master_handler->disable_log_bin_local();
     # print current_time_us() . " Drpping app user on the orig master..\n";
      print current_time_us() . " drop vip $vip..\n";
      #drop_app_user($orig_master_handler);
     &drop_vip();

      ## Waiting for N * 100 milliseconds so that current connections can exit
      my $time_until_read_only = 15;
      $_tstart = [gettimeofday];
      my @threads = get_threads_util( $orig_master_handler->{dbh},
        $orig_master_handler->{connection_id} );
      while ( $time_until_read_only > 0 && $#threads >= 0 ) {
        if ( $time_until_read_only % 5 == 0 ) {
          printf
"%s Waiting all running %d threads are disconnected.. (max %d milliseconds)\n",
            current_time_us(), $#threads + 1, $time_until_read_only * 100;
          if ( $#threads < 5 ) {
            print Data::Dumper->new( [$_] )->Indent(0)->Terse(1)->Dump . "\n"
              foreach (@threads);
          }
        }
        sleep_until();
        $_tstart = [gettimeofday];
        $time_until_read_only--;
        @threads = get_threads_util( $orig_master_handler->{dbh},
          $orig_master_handler->{connection_id} );
      }

      ## Setting read_only=1 on the current master so that nobody(except SUPER) can write
      print current_time_us() . " Set read_only=1 on the orig master.. ";
      $orig_master_handler->enable_read_only();
      if ( $orig_master_handler->is_read_only() ) {
        print "ok.\n";
      }
      else {
        die "Failed!\n";
      }

      ## Waiting for M * 100 milliseconds so that current update queries can complete
      my $time_until_kill_threads = 5;
      @threads = get_threads_util( $orig_master_handler->{dbh},
        $orig_master_handler->{connection_id} );
      while ( $time_until_kill_threads > 0 && $#threads >= 0 ) {
        if ( $time_until_kill_threads % 5 == 0 ) {
          printf
"%s Waiting all running %d queries are disconnected.. (max %d milliseconds)\n",
            current_time_us(), $#threads + 1, $time_until_kill_threads * 100;
          if ( $#threads < 5 ) {
            print Data::Dumper->new( [$_] )->Indent(0)->Terse(1)->Dump . "\n"
              foreach (@threads);
          }
        }
        sleep_until();
        $_tstart = [gettimeofday];
        $time_until_kill_threads--;
        @threads = get_threads_util( $orig_master_handler->{dbh},
          $orig_master_handler->{connection_id} );
      }

      ## Terminating all threads
      print current_time_us() . " Killing all application threads..\n";
      $orig_master_handler->kill_threads(@threads) if ( $#threads >= 0 );
      print current_time_us() . " done.\n";
      $orig_master_handler->enable_log_bin_local();
      $orig_master_handler->disconnect();

      ## After finishing the script, MHA executes FLUSH TABLES WITH READ LOCK
      $exit_code = 0;
    };
    if ($@) {
      warn "Got Error: $@\n";
      exit $exit_code;
    }
    exit $exit_code;
  }
  elsif ( $command eq "start" ) {
    ## Activating master ip on the new master
    # 1. Create app user with write privileges
    # 2. Moving backup script if needed
    # 3. Register new master's ip to the catalog database

# We don't return error even though activating updatable accounts/ip failed so that we don't interrupt slaves' recovery.
# If exit code is 0 or 10, MHA does not abort
    my $exit_code = 10;
    eval {
      my $new_master_handler = new MHA::DBHelper();

      # args: hostname, port, user, password, raise_error_or_not
      $new_master_handler->connect( $new_master_ip, $new_master_port,
        $new_master_user, $new_master_password, 1 );

      ## Set read_only=0 on the new master
      $new_master_handler->disable_log_bin_local();
      print current_time_us() . " Set read_only=0 on the new master.\n";
      $new_master_handler->disable_read_only();

      ## Creating an app user on the new master
      #print current_time_us() . " Creating app user on the new master..\n";
      print current_time_us() . "Add vip $vip on $if..\n";
     # create_app_user($new_master_handler);
      &add_vip();
      $new_master_handler->enable_log_bin_local();
      $new_master_handler->disconnect();

      ## Update master ip on the catalog database, etc
      $exit_code = 0;
    };
    if ($@) {
      warn "Got Error: $@\n";
      exit $exit_code;
    }
    exit $exit_code;
  }
  elsif ( $command eq "status" ) {

    # do nothing
    exit 0;
  }
  else {
    &usage();
    exit 1;
  }
}

sub usage {
  print
"Usage: master_ip_online_change --command=start|stop|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
  die;
}

===============================================================
[root@node1 masterha]# cat check_ssh.sh 
/usr/bin/masterha_check_ssh --global_conf=/etc/masterha/masterha_default.conf --conf=/etc/masterha/app1.conf


=========================================================================
[root@node1 masterha]# cat check_repl.sh 
/usr/bin/masterha_check_repl --global_conf=/etc/masterha/masterha_default.conf --conf=/etc/masterha/app1.conf
[root@node1 masterha]# 

=============================================================================
[root@node1 masterha]# cat start-manager.sh 
nohup /usr/bin/masterha_manager --global_conf=/etc/masterha/masterha_default.conf --conf=/etc/masterha/app1.conf  > /tmp/mha_manager.log 2>&1 &
[root@node1 masterha]# 












