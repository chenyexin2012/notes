	
## 主从复制方式

1. 异步复制：mysql默认使用异步复制的方式实现主从数据的同步，这种方式并不能保证数据传送到从库节点上。

2. 全同步复制：当主库提交事务之后，必须等待所有的从节点提交事务才会继续进行后续操作，这种方式能够保证主从数据的一致性，但是主库完成一个事务的时间被拉长，性能降低。

3. 半同步复制：这是一种介于异步和全同步复制之间的一种同步方式，当从库的IO线程接收完毕binlog日志之后，会向主库发送一个确认消息，此时主库才会提交，但并不保证后续从库能正确的将数据写入库中。如果主库超过指定时间(rpl_semi_sync_master_timeout=10000，默认10秒)未收到确认信号，会将同步方式改为异步复制方式。

### 配置过程

注：下列操作数据库版本为 5.7

#### 配置主从复制：	

主库ip：192.168.31.11
从库ip：192.168.31.12
	
1. 修改主库配置文件在[mysqld]部分新增：
	
		[mysqld]
		#开启二进制日志
		log-bin=mysql-bin 
		#设置server-id，建议设置为当前ip的最后一个段的数字,这样不会乱
		server-id=11 
	
	其它配置：
	
		# 使binlog在每N次binlog写入后与硬盘同步
		sync-binlog=1
		# 1天时间自动清理二进制日志
		expire_logs_days=1
		# 不同步哪些数据库  
		binlog-ignore-db = sys  
		binlog-ignore-db = mysql  
		binlog-ignore-db = information_schema  
		binlog-ignore-db = performation_schema  
			
		# 只同步哪些数据库，除此之外，其他不同步  
		binlog-do-db = test
	
2. 重启数据库后为从库新增用户（此处以一个从库为例）：

		mysql> CREATE USER 'slave'@'192.168.31.12' IDENTIFIED BY '123456';	#创建用户
		Query OK, 0 rows affected (0.01 sec)

		mysql> GRANT REPLICATION SLAVE ON *.* TO 'slave'@'192.168.31.12';	#分配权限
		Query OK, 0 rows affected (0.00 sec)

		mysql> flush privileges;	#刷新权限
		Query OK, 0 rows affected (0.00 sec)


3. 查看主库状态,日志文件名,日志位置(记录文件名与位置)

		mysql> show master status;
		+------------------+----------+--------------+--------------------------------------------------+-------------------+
		| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB                                 | Executed_Gtid_Set |
		+------------------+----------+--------------+--------------------------------------------------+-------------------+
		| mysql-bin.000006 |     1091 |              | sys,mysql,information_schema,performation_schema |                   |
		+------------------+----------+--------------+--------------------------------------------------+-------------------+
		1 row in set (0.00 sec)



4. 修改从库配置文件在[mysqld]部分新增：
	
		[mysqld]
		server-id=12 #设置server-id，建议设置为当前ip的最后一个段的数字,这样不会乱
	
	其它配置：
	
		# 二进制日志自动删除的天数
		expire_logs_days=1
	
5. 重启从库后，执行同步SQL并开启同步模式：

		mysql> CHANGE MASTER TO MASTER_HOST='192.168.31.11', MASTER_USER='slave', MASTER_PASSWORD='123456', MASTER_LOG_FILE='mysql-bin.000006', MASTER_LOG_POS=1091;
		Query OK, 0 rows affected (0.03 sec)

		mysql> START SLAVE;
		Query OK, 0 rows affected (0.00 sec)

	
6. 查看从库状态，当Slave_IO_Running和Slave_SQL_Running都为YES的时候就表示主从同步设置成功了。

		mysql> show slave status\G;
		*************************** 1. row ***************************
					Slave_IO_State: Waiting for master to send event
						Master_Host: 192.168.31.11
						Master_User: slave
						Master_Port: 3306
						Connect_Retry: 60
					Master_Log_File: mysql-bin.000016
				Read_Master_Log_Pos: 51483007
					Relay_Log_File: localhost-relay-bin.000002
						Relay_Log_Pos: 320
				Relay_Master_Log_File: mysql-bin.000016
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
				Exec_Master_Log_Pos: 51483007
					Relay_Log_Space: 531
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
					Master_Server_Id: 100
						Master_UUID: 9eb6cdb9-0d96-11ea-a493-000c292d527c
					Master_Info_File: /home/mysql/data/data/master.info
							SQL_Delay: 0
				SQL_Remaining_Delay: NULL
			Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
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
		1 row in set (0.00 sec)

	注：如果安装的版本为mysql 8.0，由于身份验证方式不同，可能会出现

		Last_IO_Error: error connecting to master 'slave@192.168.31.11:3306' - retry-time: 60 retries: 3 message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.

	此时主库在创建slave用户时应注意选择认证方式，如：

		CREATE USER 'slave'@'192.168.31.12' IDENTIFIED WITH mysql_native_password BY '123456';


7. 配置双主复制（如需）

	对两个数据库分别进行上述操作。

#### 配置半同步复制方式

1. 主库安装半同步复制模块

		mysql> install plugin rpl_semi_sync_master soname 'semisync_master.so';
		Query OK, 0 rows affected (0.04 sec)

2. 查看安装情况

		mysql> show global variables like "%semi%";
		+-------------------------------------------+------------+
		| Variable_name                             | Value      |
		+-------------------------------------------+------------+
		| rpl_semi_sync_master_enabled              | OFF        |
		| rpl_semi_sync_master_timeout              | 10000      |
		| rpl_semi_sync_master_trace_level          | 32         |
		| rpl_semi_sync_master_wait_for_slave_count | 1          |
		| rpl_semi_sync_master_wait_no_slave        | ON         |
		| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
		+-------------------------------------------+------------+
		6 rows in set (0.00 sec)

		rpl_semi_sync_master_enabled： 是否启用半同步复制
		rpl_semi_sync_master_timeout： 超时时间，主库超过此时间未接收到响应，改为异步复制

3. 启用半同步复制

		mysql> set global rpl_semi_sync_master_enabled = 1;
		Query OK, 0 rows affected (0.00 sec)

		mysql> show global variables like "%semi%";
		+-------------------------------------------+------------+
		| Variable_name                             | Value      |
		+-------------------------------------------+------------+
		| rpl_semi_sync_master_enabled              | ON         |
		| rpl_semi_sync_master_timeout              | 10000      |
		| rpl_semi_sync_master_trace_level          | 32         |
		| rpl_semi_sync_master_wait_for_slave_count | 1          |
		| rpl_semi_sync_master_wait_no_slave        | ON         |
		| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
		+-------------------------------------------+------------+
		6 rows in set (0.01 sec)

以上操作也可写入配置文件中，保证永久生效：

		[mysqld]
		rpl_semi_sync_master_enabled = 1;
		rpl_semi_sync_master_timeout = 10000;


4. 从库安装半同步复制模块并启用半同步复制

		mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
		Query OK, 0 rows affected (0.02 sec)

		mysql> set global rpl_semi_sync_slave_enabled = 1;
		Query OK, 0 rows affected (0.00 sec)

		mysql> show global variables like "%semi%";
		+---------------------------------+-------+
		| Variable_name                   | Value |
		+---------------------------------+-------+
		| rpl_semi_sync_slave_enabled     | ON    |
		| rpl_semi_sync_slave_trace_level | 32    |
		+---------------------------------+-------+
		2 rows in set (0.00 sec)

5. 重启io线程

		mysql> stop slave io_thread;
		Query OK, 0 rows affected (0.01 sec)

		mysql> start slave io_thread;
		Query OK, 0 rows affected (0.00 sec)

6. 在主库上查看是否启用成功

		mysql> show global status like "rpl%";
		+--------------------------------------------+-------+
		| Variable_name                              | Value |
		+--------------------------------------------+-------+
		| Rpl_semi_sync_master_clients               | 1     |
		| Rpl_semi_sync_master_net_avg_wait_time     | 0     |
		| Rpl_semi_sync_master_net_wait_time         | 0     |
		| Rpl_semi_sync_master_net_waits             | 0     |
		| Rpl_semi_sync_master_no_times              | 1     |
		| Rpl_semi_sync_master_no_tx                 | 5     |
		| Rpl_semi_sync_master_status                | ON    |
		| Rpl_semi_sync_master_timefunc_failures     | 0     |
		| Rpl_semi_sync_master_tx_avg_wait_time      | 0     |
		| Rpl_semi_sync_master_tx_wait_time          | 0     |
		| Rpl_semi_sync_master_tx_waits              | 0     |
		| Rpl_semi_sync_master_wait_pos_backtraverse | 0     |
		| Rpl_semi_sync_master_wait_sessions         | 0     |
		| Rpl_semi_sync_master_yes_tx                | 0     |
		+--------------------------------------------+-------+
		14 rows in set (0.00 sec)

		Rpl_semi_sync_master_clients !=0 and Rpl_semi_sync_master_status = ON