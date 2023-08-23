

vmware 克隆之后，进行主从数据库的配置

只要在文件里声明的环境变量，都可以克隆软件成功



成功了java，git, maven



### my.cnf

mysql配置文件

```sh
find / -name my.cnf
```

   显示有多个my.cnf文件



my.cnf文件加载顺序

```sh
[root@localhost etc]# mysql --help|grep 'my.cnf'
                      order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf 
sh
```

从头开始加载，优先级逐渐上升。

~：代表用户目录/home/wcy

### auto.cnf

与my.cnf文件  的查找，及加载顺序查看命令类似

里边有uuid，因为是克隆的虚拟机，所以2者的uuid相同，需修改。

### notes

更新配置文件后，需重启mysql，

```sh
service mysql restart
```

两台虚拟机通信，关闭防火墙

```sh
#查看防火墙状态
systemctl status firewalld
service  iptables status
#暂时关闭防火墙
systemctl stop firewalld
service  iptables stop
#永久关闭防火墙
systemctl disable firewalld
chkconfig iptables off
#重启防火墙
systemctl enable firewalld
service iptables restart  

```



### master配置



my.cnf

```sh

[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove the leading "# " to disable binary logging
# Binary logging captures changes between backups and is enabled by
# default. It's default setting is log_bin=binlog
# disable_log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
#
# Remove leading # to revert to previous value for default_authentication_plugin,
# this will increase compatibility with older clients. For background, see:
# https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_default_authentication_plugin
# default-authentication-plugin=mysql_native_password
character_set_server=utf8
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
server-id=1
log-bin=mysql-bin
binlog-ignore-db=mysql
binlog-ignore-db=infomation_schema
binlog-do-db=reggie
binlog_format=STATEMENT
```



```sh
mysql -u root -p
```

[MySQL 8.0及更高版本不支持在`GRANT`语句中使用`IDENTIFIED BY`子句](https://stackoverflow.com/questions/41960979/how-to-grant-replication-privilege-to-a-database-in-mysql)[1](https://stackoverflow.com/questions/41960979/how-to-grant-replication-privilege-to-a-database-in-mysql)[。您需要分别执行`CREATE USER`语句和`GRANT`语句](https://dba.stackexchange.com/questions/174842/mysql-replication-slave-cant-connect-to-master)[2](https://dba.stackexchange.com/questions/174842/mysql-replication-slave-cant-connect-to-master)

```mysql
update user set host='%' where user='root';

CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%';


mysql> show master status;
+------------------+----------+--------------+-------------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB        | Executed_Gtid_Set |
+------------------+----------+--------------+-------------------------+-------------------+
| mysql-bin.000001 |      156 | reggie       | mysql,infomation_schema |                   |
+------------------+----------+--------------+-------------------------+-------------------+
1 row in set (0.00 sec)

mysql> show variables like '%server_id%';
+----------------+-------+
| Variable_name  | Value |
+----------------+-------+
| server_id      | 1     |
| server_id_bits | 32    |
+----------------+-------+
2 rows in set (0.12 sec)


mysql> show variables like '%server_uuid%';
+---------------+--------------------------------------+
| Variable_name | Value                                |
+---------------+--------------------------------------+
| server_uuid   | f9da5fde-4106-11ee-ab13-000c292c6476 |
+---------------+--------------------------------------+
```



### slave配置

```sh
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html

[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove the leading "# " to disable binary logging
# Binary logging captures changes between backups and is enabled by
# default. It's default setting is log_bin=binlog
# disable_log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M

# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
#
# Remove leading # to revert to previous value for default_authentication_plugin,
# this will increase compatibility with older clients. For background, see:
# https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_default_authentication_plugin
# default-authentication-plugin=mysql_native_password
character_set_server=utf8
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
server-id=2
relay-log=mysql-relay
```

配置命令

```mysql

stop slave;
reset slave all;
start slave;

change master to master_host='192.168.188.130', master_user='slave', master_password='123456', master_log_file='mysql-bin.000001', master_log_pos=156, get_master_public_key=1;

show slave status\G;
```

```mysql
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 192.168.188.130
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 156
               Relay_Log_File: mysql-relay.000002
                Relay_Log_Pos: 324
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 156
              Relay_Log_Space: 529
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: f9da5fde-4106-11ee-ab13-000c292c6476
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
       Master_public_key_path: 
        Get_master_public_key: 1
            Network_Namespace: 
1 row in set, 1 warning (0.00 sec)

ERROR: 
No query specified
```

### 遇到的问题

1. 密码配置中不能设置简单的密码，更改密码策略也没有奏效

- 结束实验后，猜测可能是 没有重启mysql服务，也不是很成立。

- 还遇到了从节点密码sha验证的问题，

  ```
  Last_IO_Error: error connecting to master 'slave@192.168.188.130:3306' - retry-time: 60 retries: 1 message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.
  ```

  new bing :

  要在主库上将从库用户的身份验证插件更改为mysql_native_password，并重新设置密码，您可以按照以下步骤操作：

  - 登录到主库的MySQL终端，使用root用户或具有足够权限的用户。
  - 执行以下SQL语句，将从库用户的身份验证插件更改为mysql_native_password，并重新设置密码。请将`slave`和`123456`替换为您实际的从库用户名和密码。

  ```
  mysql> ALTER USER 'slave'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
  ```

  复制

  - 执行以下SQL语句，刷新权限。

  ```
  mysql> FLUSH PRIVILEGES;
  ```

  复制

  - 退出MySQL终端。
  - 在从库上，重新启动复制过程，或者重新执行CHANGE REPLICATION SOURCE TO语句，指定从库用户和密码。

  [这样就可以使用mysql_native_password身份验证插件连接到主库了。但是，请注意，这种方法会降低安全性，因为密码会以明文形式传输。如果您想使用更安全的caching_sha2_password身份验证插件，您可以参考以下链接](https://stackoverflow.com/questions/49931541/mysql-changing-authentication-type-from-standard-to-caching-sha2-password)[1](https://stackoverflow.com/questions/49931541/mysql-changing-authentication-type-from-standard-to-caching-sha2-password)

2. 配置了mysql的主从节点后，主节点可写，会同步到从节点，是正常的，但是通过navicat连接的从库也可以写，没有同步到主库，按理从节点不能写，可能在spring中配置了从节点，就不会再把写请求发到从节点了。

