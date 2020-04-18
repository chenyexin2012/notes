## centos安装docker

1. 使用命令 uname -a 查看linux内核，docker至少需要3.8以上

    [root@localhost ~]# uname -a
    Linux localhost.localdomain 3.10.0-1062.4.1.el7.x86_64 #1 SMP Fri Oct 18 17:15:30 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

2. 使用 yum update 升级yum

3. 安装所需软件包

    yum install -y yum-utils device-mapper-persistent-data lvm2

4. 设置仓库

    ## 使用阿里云镜像源
    yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

5. 更新并安装 Docker-CE

    yum makecache fast
    yum -y install docker-ce

6. 启动

    service docker start

其它命令：

    ## 查看版本
    docker version
    ## 查找镜像
    docker search XXX
    ## 拉取镜像
    docker pull XXX
    ## 查看镜像
    docker images
    ## 查看运行中的镜像
    docker ps

## 使用docker安装mysql

1. 拉取mysql镜像，默认拉取最新版本的镜像

    docker pull mysql

2. 运行mysql

    docker run --name mysql-test -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql

    参数说明：
    --name mysql-test:：
    -p 3306:3306：映射容器服务的3306(后)端口到宿主机的3306端口，外部主机可以直接通过宿主机的3306端口访问MySQL。 
    -e MYSQL_ROOT_PASSWORD=123456：初始化root用户的密码。
    -d: 后台运行容器，并返回容器ID。

3. 查看是否启动成功

    input:
    docker ps | grep mysql

    output:
    52808ce0a4d4        mysql               "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes        0.0.0.0:3306->3306/tcp, 33060/tcp   mysql-test


4. 进入容器
    ## 使用创建时指定的容器名
    docker exec -it mysql-test bash

5. Mysql相关操作参考mysql安装过程

    [mysql安装过程](/mysql/centos安装mysql过程参考.md)

6. 其它命令

    ## 停止容器
    docker top XXX
    ## 删除容器
    docker rm XXX