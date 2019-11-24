	
## 修改主库：	
	
1. 修改主库配置文件在[mysqld]部分新增：
	
		[mysqld]
		#开启二进制日志
		log-bin=mysql-bin 
		#设置server-id，建议设置为当前ip的最后一个段的数字,这样不会乱
		server-id=100 
	
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

		CREATE USER 'slave'@'192.168.227.129' IDENTIFIED BY '123456';	#创建用户
		GRANT REPLICATION SLAVE ON *.* TO 'slave'@'192.168.227.129';	#分配权限
		flush privileges;	#刷新权限

3. 查看主库状态,日志文件名,日志位置(记录文件名与位置)

	input: 
	
		show master status;
	
	output:

		+------------------+----------+--------------+------------------+-------------------+
		| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
		+------------------+----------+--------------+------------------+-------------------+
		| mysql-bin.000001 |      769 |              |                  |                   |
		+------------------+----------+--------------+------------------+-------------------+


## 修改从库：

1. 修改从库配置文件在[mysqld]部分新增：
	
		[mysqld]
		server-id=129 #设置server-id，建议设置为当前ip的最后一个段的数字,这样不会乱
	
	其它配置：
	
		# 二进制日志自动删除的天数
		expire_logs_days=1
	
2. 重启从库后，执行同步SQL并开启同步模式：

		CHANGE MASTER TO MASTER_HOST='192.168.227.129', MASTER_USER='slave', MASTER_PASSWORD='123456', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=769;
		START SLAVE;
	
3. 查看从库状态，当Slave_IO_Running和Slave_SQL_Running都为YES的时候就表示主从同步设置成功了。

	input: 
	
		show slave status\G;
	
	output: 
	
				*************************** 1. row ***************************
						Slave_IO_State: Waiting for master to send event
							Master_Host: 192.168.227.129
							Master_User: slave
							Master_Port: 3306
						Connect_Retry: 60
						Master_Log_File: mysql-bin.000001
					Read_Master_Log_Pos: 769
						Relay_Log_File: localhost-relay-bin.000002
						Relay_Log_Pos: 320
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
					Exec_Master_Log_Pos: 769
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
						Master_Info_File: /home/mysql/data-slave/data/master.info
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
