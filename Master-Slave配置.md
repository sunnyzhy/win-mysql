# 修改主服务器的配置文件my.cnf
```
# vim /etc/my.cnf
[mysqld]
server-id=200 #取本机ip的末端即可
binlog-do-db=db_name #要同步的数据库
#binlog-ignore-db=db_name   #不同步的数据库,如果指定了binlog-do-db这里应该可以不用指定的
log-bin=mysql-bin #开启二进制日志，这一点决定了数据同步的成败，mysql-bin是自定义的二进制日志名称
```
    开启二进制日志后，每次开启数据库都会在\mysql\data下自动生成一个mysql-log.000001，按照顺序，每次重启数据库会生成这样的一个二进制日志文件，同时还会自动生成一个mysql-log.index文件。

# 在主库添加用户并授权
```
mysql> use mysql;

mysql> create user 'username'@'slave ip' identified by 'password';

mysql> grant replication slave on *.* to 'username'@'slave ip';

mysql> select * from user where user = 'username' \G;
Repl_slave_priv: Y
```

# 删除用户及授权
```
mysql> revoke all on *.* from 'username'@'slave ip'; #删除授权

mysql> use mysql;

mysql> delete from user where user='username' and host='slave ip' ; #删除用户

mysql> flush privileges; #刷新
```

# 记录主库File和Position项对应的值
```
mysql> show master status \G;
File: mysql-bin.000001
Position: 335
```

# 修改从服务器的配置文件my.cnf
```
# vim /etc/my.cnf
[mysqld]
server-id=210
#replicate-wild-do-table=db_name.table_name
replicate-do-db=db_name
log-bin=mysql-bin
```

# 配置从库连接主库所需的信息
```
mysql> stop slave;

mysql> change master to master_host='master ip',master_port=3306,master_user='username',master_password='password',master_log_file='mysql-bin.000001',master_log_pos=335;
```
    master_log_file对应主库File项值
    master_log_pos对应主库Position项值

# 开启从库的同步功能
```
mysql> start slave;
```

# 查看从库状态
```
mysql> show slave status \G;
Slave_IO_State: Waiting for master to send event
Slave_IO_Running: Yes  #连接到主库，并读取主库的日志到本地，生成本地日志文件
Slave_SQL_Running: Yes #读取本地日志文件，并执行日志里的SQL命令
Master_Log_File: mysql-bin.000001 #主库file项对应的值
Read_Master_Log_Pos: 335 #主库position项对应的值
```
    从库配置好之后，在\mysql\data下会自动生成如下文件：master.info和relay-log.info

# 在主库执行CRUD操作，从库会同步更新

**注意：**
- 主从库间的数据不是实时同步的。 

- 如果主从库之间的网络断开，则从库会在网络正常后批量同步。 

- 如果在从库修改数据，就很可能造成从库在执行主库的bin-log时出现错误而停止同步，一般不建议在从库进行CRUD操作。
