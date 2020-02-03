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


## 客户端工具zkCli常用命令

zookeeper客户端工具在安装目录的bin文件夹下。

### 连接命令

    zkCli.sh -server localhost:2181


### 查看命令

查看根目录节点： ls /

查看根节点详细数据：ls -s /

    czxid - 创建节点的事务zxid
    每次修改ZooKeeper状态都会收到一个zxid形式的时间戳，也就是ZooKeeper事务ID。
    事务ID是ZooKeeper中所有修改总的次序。每个修改都有唯一的zxid，如果zxid1小于zxid2，那么zxid1在zxid2之前发生。
    ctime - znode被创建的毫秒数(从1970年开始)
    mzxid - znode最后更新的事务zxid
    mtime - znode最后修改的毫秒数(从1970年开始)
    pZxid-znode最后更新的子节点zxid
    cversion - znode子节点变化号，znode子节点修改次数
    dataversion - znode数据变化号
    aclVersion - znode访问控制列表的变化号
    ephemeralOwner- 如果是临时节点，这个是znode拥有者的session id。如果不是临时节点则是0。
    dataLength- znode的数据长度
    numChildren - znode子节点数量

查看根节点状态：stat /

查看根节点数据和详情：get /

查看指定节点，如：get /node1/node11

### 创建节点

在根目录下创建节点node1，内容为hello：

    create /node1 hello

在node1节点下新增节点node11，内容为空：

    create /node1/node11

创建临时节点，加上 -e 参数，客户端断开后会自动删除，如：

    create -e /node2

创建带序号的节点，加上 -s 参数，如：

    create -s /node

运行结果为：
    
    Created /node0000000004

序号为4，说明之前已经创建过节点，没有则从0开始。

### 修改节点

修改节点 /node1 为 HELLO:

    set /node1 HELLO

运行 get /node1 查看结果：

    HELLO

### 删除节点

删除节点 /node1：

    delete /node1

若 /node1 存在子节点，则返回：

    Node not empty: /node1

需要使用递归删除，如下：

    deleteall /node1

### 监听节点的变化

监听节点的值变化：

    get -s -w /node

返回值如下：

    node12
    cZxid = 0x7d
    ctime = Mon Feb 03 12:24:16 CST 2020
    mZxid = 0x7f
    mtime = Mon Feb 03 12:25:43 CST 2020
    pZxid = 0x7d
    cversion = 0
    dataVersion = 2
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 6
    numChildren = 0

当节点值发生变化时，输出（只能监听一次）：

    WATCHER::

    WatchedEvent state:SyncConnected type:NodeDataChanged path:/node

监听节点的子节点变化：

    ls -w /node

输出如下：

    []

当其它客户端在 node 下增删节点时，输出（只能监听一次）：

    WATCHER::

    WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/node
