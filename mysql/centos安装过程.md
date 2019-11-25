## centos安装mysql-5.7.25，使用普通用户权限，此处使用mysql用户

1. 在镜像网站下载安装文件，如mysql-5.7.25-linux-glibc2.12-x86_64.tar.gz

2. 解压文件至 /home/mysql/services 文件夹下，重命名为mysql-5.7.25，使用chown -R 命令修改权限，切换至用户mysql

3. 修改配置文件 [my.cnf](my.cnf)

4. 使用初始化命令 

		./bin/mysqld --defaults-file=/home/mysql/services/mysql-slave-5.7.25/my.cnf --initialize --user=mysql --datadir=/home/mysql/data-slave/data --basedir=/home/mysql/services/mysql-slave-5.7.25

5. 查看日志获取初始密码

		2019-11-23T02:11:54.727262Z 1 [Note] A temporary password is generated for root@localhost: Z2.f1ji;lS)s

6. 启动服务，不指定配置路径则默认为/etc/my.cnf

		./bin/mysqld_safe --defaults-file=/home/mysql/services/mysql-slave-5.7.25/my.cnf &

7. 使用默认密码登录数据库： ./bin/mysql -P3306 -uroot -p

	可能出现的问题：
	
		error: 'Can't connect to local MySQL server through socket '/tmp/mysql.sock'
		
		对mysql.sock这个文件来说，其作用是程序与mysqlserver处于同一台机器，发起本地连接时可用。 
		
		因此需要加上host地址： ./bin/mysql -h127.0.0.1 -P3306 -uroot -p
		
8. 修改数据库密码：
	
		update mysql.user set authentication_string=password('123456') where user='root';
	
	这种方式可能会出现：ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
	这是数据库默认第一次登录必须修改默认密码导致。

	因此需要使用：

		alter user 'root'@'localhost' identified by '123456';  
		或者 set password=password("123456");

		flush privileges; # 刷新权限信息（可省略）
	
9. 可以使用以下命令关闭数据库：

		./bin/mysqladmin shutdown -h127.0.0.1 -uroot -p

10. 开放远程登录权限：

		update mysql.user set host = '%' where user = 'root';
		flush privileges;

11. 开放端口号（以下操作使用root用户）：  

		开放3306端口：firewall-cmd --zone=public --add-port=3306/tcp --permanent
		重启：firewall-cmd --reload
		查询：firewall-cmd --query-port=3306/tcp

12. 使用systemctl设置开机自启：

	进入目录/usr/lib/systemd/system，添加文件[mysql.service](mysql.service)

	