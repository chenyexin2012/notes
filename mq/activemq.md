## centos安装Active MQ

    wget https://mirrors.tuna.tsinghua.edu.cn/apache/activemq/5.15.10/apache-activemq-5.15.10-bin.tar.gz

    直接解压安装，进入bin文件夹下

    命令： ./activemq start |stop |restart |status

    开放8161（管理平台）端口：firewall-cmd --zone=public --add-port=8161/tcp --permanent
    开放61616（连接）端口：firewall-cmd --zone=public --add-port=61616/tcp --permanent
    重启：firewall-cmd --reload

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


