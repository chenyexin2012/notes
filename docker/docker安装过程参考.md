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
    ## 删除镜像
    docker rmi XXX

docker run 命令说明:

    -a stdin: 指定标准输入输出内容类型，可选 STDIN/STDOUT/STDERR 三项；
    -d: 后台运行容器，并返回容器ID；
    -i: 以交互模式运行容器，通常与 -t 同时使用；
    -P: 随机端口映射，容器内部端口随机映射到主机的端口
    -p: 指定端口映射，格式为：主机(宿主)端口:容器端口
    -t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
    --name="nginx-lb": 为容器指定一个名称；
    --dns 8.8.8.8: 指定容器使用的DNS服务器，默认和宿主一致；
    --dns-search example.com: 指定容器DNS搜索域名，默认和宿主一致；
    -h "mars": 指定容器的hostname；
    -e username="ritchie": 设置环境变量；
    --env-file=[]: 从指定文件读入环境变量；
    --cpuset="0-2" or --cpuset="0,1,2": 绑定容器到指定CPU运行；
    -m :设置容器使用内存最大值；
    --net="bridge": 指定容器的网络连接类型，支持 bridge/host/none/container: 四种类型；
    --link=[]: 添加链接到另一个容器；
    --expose=[]: 开放一个端口或一组端口；
    --volume , -v: 绑定一个卷


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
    ## 停止所有容器
    docker stop $(docker ps -a -q)
    ## 删除所有容器
    docker rm $(docker ps -a -q)


## 使用docker部署jar

1. 在jar同级目录下创建 Dockerfile 文件 

    FROM java:8
    MAINTAINER holmes
    COPY test.jar test.jar
    ENTRYPOINT ["java","-jar","test.jar"]

[Dockerfile文件参数说明](Dockerfile)

2. 通过 Dockerfile 文件生成 镜像

    docker build -t test .

    说明：test表示为镜像取得名，后面的 . 不能省略，表示使用当前目录的 Dockerfile 文件

3. 通过 docker images 命令可以查看生成的镜像

    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    test                latest              7e7a2f70ce4b        25 seconds ago      688MB

4. 创建容器并运行

    docker -d -p 6001:6001 --name test test

5. 使用 docker ps 命令可以查看运行的容器

    CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                               NAMES
    96144f63da62        test                "java -jar test.jar"   37 seconds ago      Up 36 seconds       0.0.0.0:6001->6001/tcp, 11111/tcp   hungry_antonelli

6. 使用 docker logs -f test 可以查看运行日志


## 使用 maven 打包 springboot项目 docker 镜像

1. docker开启远程镜像提交功能

    修改 docker.service 文件, 在 ExecStart 启动参数 /usr/bin/dockerd 后加上如下内容: 

        -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock


2. 添加依赖

        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>com.holmes.springcloud.eureka.EurekaSpringApplication</mainClass>
                </configuration>
            </plugin>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.2.2</version>
                <configuration>
                    <imageName>eureka</imageName>
                    <dockerHost>http://peer1:2375</dockerHost>
                    <baseImage>java</baseImage>
                    <maintainer>holmes</maintainer>
                    <cmd>["java", "-version"]</cmd>
                    <entryPoint>["java", "-jar", "${project.build.finalName}.jar"]</entryPoint>
                    <!-- 这里是复制 jar 包到 docker 容器指定目录配置 -->
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>

3. 运行命令进行打包

    mvn clean package docker:build -Dmaven.test.skip=true

4. 在docker中运行 docker images 查看镜像


## 搭建私有仓库 Registry

1. 下载Registry镜像并启动

    # 拉取镜像
    docker pull registry

    # 启动镜像
    # -v /data/registry:/var/lib/registry 将宿主机的/data/registry目录绑定到容器/var/lib/registry目录，这个目录用于存放镜像
    docker run -itd -v /data/registry:/var/lib/registry -p 5000:5000 --restart=always --name registry registry:latest 
    
    # 访问路径http://127.0.0.1:5000/v2/_catalog
    [root@peer1 opt]# curl http://127.0.0.1:5000/v2/_catalog
    {"repositories":[]}

2. 测试仓库（将 registry 镜像推入仓库）

    # 为镜像打标签，registry:latest 为需要上传的镜像，peer1:5000/registry:m 为目标镜像
    docker tag registry:latest peer1:5000/registry:m

    # 将镜像上传至仓库
    docker push peer1:5000/registry:my

    注：若出现 “Get https://127.0.0.1:5000/v2/: http: server gave HTTP response to HTTPS client” 说明出现了错误，这是因为上传需要使用 https，此时需要修改 /etc/docker/daemon.json 文件，加入 "insecure-registries": [ "peer1:5000"]，之后重启docker。

    {
        "registry-mirrors": ["https://3laho3y3.mirror.aliyuncs.com"],
        "insecure-registries": [ "peer1:5000"]
    }

    # 查看所有镜像
    curl http://peer1:5000/v2/_catalog

    # 列出registry镜像的tag
    curl http://peer1:5000/v2/registry/tags/list

    # 拉取上传的镜像
    docker pull peer1:5000/registry:my


## 安装图形化管理界面 shipyard

1. 下载基础镜像

    docker pull alpine
    docker pull rethinkdb
    docker pull microbox/etcd
    docker pull ehazlett/curl
    docker pull rancher/rancher-agent
    docker pull rancher/rancher
    docker pull shipyard/docker-proxy
    docker pull swarm 
    docker pull shipyard/shipyard
    ## 中文镜像
    docker pull dockerclub/shipyard

2. 一键部署脚本

[英文部署脚本](shipyard-deploy-en)
[中文部署脚本](shipyard-deploy-cn)