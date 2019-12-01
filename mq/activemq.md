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


## 如何保证顺序消费

### 使用独占消费者（exclusive consumer）

ActiveMQ从4.X版本开始支持ExclusiveConsumer（或者说是Exclusive Queue），Broker会从多个Consumer中挑选一个Consumer来处理所有的消息，从而保证消息的有序处理。一旦这个Consumer失效，Broker会自动切换到其他的Consumer来消费。

可以在消费端通过Destination的Option来创建一个Exclusive Consumer，例如：

    ActiveMQQueue queue = new ActiveMQQueue("TEST.QUEUE?consumer.exclusive=true");

还可以设置优先级针对网络情况进行优化，如下：

    ActiveMQQueue queue = new ActiveMQQueue("TEST.QUEUE?consumer.exclusive=true&consumer.priority=10");

### 消息分组(massage groups)

在Message对象上设置JMSXGroupID属性来对消息进行分组。Message Groups这种特性保证具有相同JMSXGroupID的消息能够被同一个活跃的消费者消费，这同时也是一种负载均衡的机制。也可以通过设置JMSXGroupSeq来关闭分组。

例如：

    Message message = session.createTextMessage("hello,world");
    message.setStringProperty("JMSXGroupID","GroupA");
    //message.setIntProperty("JMSXGroupSeq", -1);

如果使用JmsMessagingTemplate，可以通过如下设置:

        Map<String, Object> headers = new HashMap<>();
        headers.put("JMSXGroupID", "groupA");
        //headers.put("JMSXGroupSeq", "-1");
        jmsTemplate.convertAndSend(TOPIC, "text message", headers);


### 消息确认机制

Activemq消息确认机制有五种类型：

    SESSION_TRANSACTED=0：事务提交并确认
    AUTO_ACKNOWLEDGE=1 ：自动确认（默认）
    CLIENT_ACKNOWLEDGE=2：客户端手动确认 
    UPS_OK_ACKNOWLEDGE=3： 自动批量确认
    INDIVIDUAL_ACKNOWLEDGE=4：单条消息确认（Activemq独有）

