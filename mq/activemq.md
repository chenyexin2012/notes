## centos安装Active MQ

    wget https://mirrors.tuna.tsinghua.edu.cn/apache/activemq/5.15.10/apache-activemq-5.15.10-bin.tar.gz

    直接解压安装，进入bin文件夹下

    命令： ./activemq start |stop |restart |status

    开放8161（管理平台）端口：firewall-cmd --zone=public --add-port=8161/tcp --permanent
    开放61616（连接）端口：firewall-cmd --zone=public --add-port=61616/tcp --permanent
    重启：firewall-cmd --reload

    开机自启设置：在目录 /usr/lib/systemd/system 下添加文件 activemq.service，使用systemctl命令进行相关设置

[参考防火墙相关命令](https://blog.csdn.net/u014079773/article/details/79745819)

[参考systemctl相关命令](https://blog.csdn.net/qq_23587541/article/details/82849480)

## 主要配置文件说明（conf文件夹下）




## 配置用户访问账号和密码


打开activemq.xml，在broker下添加：

    <!-- 添加访问ActiveMQ的账号密码 -->
    <plugins>
        <simpleAuthenticationPlugin>
            <users>
                <authenticationUser username="holmes" password="123456" groups="users,admins"/>
            </users>
        </simpleAuthenticationPlugin>
    </plugins>


## 配置管理页面登录账号和密码

打开jetty.xml，将authenticate设置为true：

    <bean id="securityConstraint" class="org.eclipse.jetty.util.security.Constraint">
        <property name="name" value="BASIC" />
        <property name="roles" value="user,admin" />
        <!-- set authenticate=false to disable login -->
        <property name="authenticate" value="true" />
    </bean>

登录名和密码保存在jetty-realm.properties文件中：

    # Defines users that can access the web (console, demo, etc.)
    # username: password [,rolename ...]
    admin: admin, admin
    user: user, user


## 持久化配置

### 默认为KahaDB存储

activemq.xml，broker节点下：

        <persistenceAdapter>
            <kahaDB directory="${activemq.data}/kahadb"/>
        </persistenceAdapter>


### JDBC存储

使用JDBC持久化方式，数据库会默认生成以下几张表：

    activemq_msgs：queue和topic的消息都存在这个表中
    activemq_acks：存储持久订阅的信息和最后一个持久订阅接收的消息ID
    activemq_lock：跟kahadb的lock文件类似，确保数据库在某一时刻只有一个broker在访问

activemq.xml，broker节点下：

    <persistenceAdapter>    
        <jdbcPersistenceAdapter dataSource="mysqlDataSource" createTablesOnStartup="true" /> 
    </persistenceAdapter>

    <bean id="mysqlDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close"> 
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>      
        <property name="url" value="jdbc:mysql://192.168.227.129:3306/activemq?relaxAutoCommit=true"/>      
        <property name="username" value="root"/>     
        <property name="password" value="123456"/>   
    </bean>

在lib文件夹下添加所需依赖的jar：

    commons-dbcp-1.4
    mysql-connector-java-8.0.9
    commons-pool-1.6 

