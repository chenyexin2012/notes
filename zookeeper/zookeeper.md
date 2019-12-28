## centos下安装zookeeper

    wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.5.6/apache-zookeeper-3.5.6-bin.tar.gz

    注意：从3.5.5版本开始需要下载带有bin名称的包，普通的包只含有源码。

    解压文件，进入conf路径，复制配置文件zoo_sample.cfg，命名为zoo.cfg，按照需求进行配置。

    进入bin路径，运行命令 ./zkServer.sh start ../conf/zoo.cfg 启动服务。

## 搭建zookeeper集群（单机伪集群）

建立三个配置文件，zoo-2181.cfg、zoo-2182.cfg、zoo-2183.cfg

    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=/home/zookeeper/data-2181
    clientPort=2181
    dataLogDir=/home/zookeeper/log-2181
    server.1=localhost:2287:3387
    server.2=localhost:2288:3388
    server.3=localhost:2289:3389

    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=/home/zookeeper/data-2182
    clientPort=2182
    dataLogDir=/home/zookeeper/log-2182
    server.1=localhost:2287:3387
    server.2=localhost:2288:3388
    server.3=localhost:2289:3389

    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=/home/zookeeper/data-2183
    clientPort=2183
    dataLogDir=/home/zookeeper/log-2183
    server.1=localhost:2287:3387
    server.2=localhost:2288:3388
    server.3=localhost:2289:3389

在配置的dataDir文件夹下，新增文件 myid，内容分别为 1、2、3，对应 server.1、server.2、server.3。

运行命令启动服务：

    ./zkServer.sh start ../conf/zoo-2181.cfg
    ./zkServer.sh start ../conf/zoo-2182.cfg
    ./zkServer.sh start ../conf/zoo-2183.cfg






