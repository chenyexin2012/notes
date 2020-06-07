## centos安装rocketmq

1. 下载安装文件：

        wget https://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.7.0/rocketmq-all-4.7.0-bin-release.zip

2. 解压后进入，bin目录下：


3. 启动mqnamesrv

        ./mqnamesrv > namesrv.log &

4. 启动mqbroker

        ./mqbroker -c ../conf/broker.conf -n peer1:9876  >  broker.log &

    注：如果出现内存不足，可修改runbroker.sh和runserver.sh调低内存

5. 关闭命令
        
        # 关闭namesrv服务
        ./mqshutdown namesrv
        # 关闭broker服务
        ./mqshutdown broker

6. 配置开机自启

启动脚本

        #!/bin/bash
        export JAVA_HOME=/usr/local/java/jdk1.8.0_221   #必须得加上这个才行
        nohup /bin/sh /usr/local/rocketmq/bin/mqnamesrv >> /usr/local/rocketmq/bin/namesrv.log &
        nohup /bin/sh /usr/local/rocketmq/bin/mqbroker -c /usr/local/rocketmq/conf/broker.conf -n peer1:9876  >>  /usr/local/rocketmq/broker.log &

关闭脚本

        #!/bin/bash
        export JAVA_HOME=/usr/local/java/jdk1.8.0_221   #必须得加上这个才行
        nohup /bin/sh /usr/local/rocketmq/bin/mqshutdown broker >>/dev/null
        nohup /bin/sh /usr/local/rocketmq/bin/mqshutdown namesrv >>/dev/null

[service文件](rocketmq.service)


## 集群搭建

### 单Master模式

只部署一台服务，风险较大，一旦Broker重启或者宕机时，会导致整个服务不可用。

### 多Master模式

一个集群无Slave，全是Master，例如2个Master或者3个Master，这种模式的优缺点如下：

- 优点：配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢），性能最高；

- 缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到影响。

部署过程同单机部署相似，如果存在多个namesrv（一般为3个），需要在broker启动时指定地址列表，以分号分隔，参考命令如下：

        ./mqbroker -c ../conf/broker.conf -n "peer1:9876;peer2:9876;peer3:9876"  >  broker.log &

Master-A配置参考：

        brokerClusterName=DefaultCluster
        brokerName=broker-a
        brokerId=0
        deleteWhen=04
        fileReservedTime=48
        brokerRole=ASYNC_MASTER
        flushDiskType=ASYNC_FLUSH

Master-B配置参考：

        brokerClusterName=DefaultCluster
        brokerName=broker-b
        brokerId=0
        deleteWhen=04
        fileReservedTime=48
        brokerRole=ASYNC_MASTER
        flushDiskType=ASYNC_FLUSH


### 多Master多Slave模式-异步复制

每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟（毫秒级），这种模式的优缺点如下：

- 优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，同时Master宕机后，消费者仍然可以从Slave消费，而且此过程对应用透明，不需要人工干预，性能同多Master模式几乎一样；

- 缺点：Master宕机，磁盘损坏情况下会丢失少量消息。

Master-A配置参考：

        brokerClusterName=DefaultCluster
        brokerName=broker-a
        brokerId=0
        deleteWhen=04
        fileReservedTime=48
        brokerRole=ASYNC_MASTER
        flushDiskType=ASYNC_FLUSH

Slave-A配置参考：

        brokerClusterName=DefaultCluster
        brokerName=broker-a
        brokerId=1
        deleteWhen=04
        fileReservedTime=48
        brokerRole=SLAVE
        flushDiskType=ASYNC_FLUSH

Master-B配置参考：

        brokerClusterName=DefaultCluster
        brokerName=broker-b
        brokerId=0
        deleteWhen=04
        fileReservedTime=48
        brokerRole=ASYNC_MASTER
        flushDiskType=ASYNC_FLUSH


Slave-B配置参考：

        brokerClusterName=DefaultCluster
        brokerName=broker-b
        brokerId=1
        deleteWhen=04
        fileReservedTime=48
        brokerRole=SLAVE
        flushDiskType=ASYNC_FLUSH

配置说明：

        1. Master与Slave通过配置相同的brokerName进行配对，Master的brokerId必须是0，Slave的BrokerId必须是大于0的数。
        2. 同一个Master下可挂载多个Slave，不同的Slave之间通过brokerId进行区分。


### 多Master多Slave模式-同步双写

每个Master配置一个Slave，有多对Master-Slave，HA采用同步双写方式，即只有主备都写成功，才向应用返回成功，这种模式的优缺点如下：

- 优点：数据与服务都无单点故障，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高；

- 缺点：性能比异步复制模式略低（大约低10%左右），发送单个消息的RT会略高，且目前版本在主节点宕机后，备机不能自动切换为主机。

配置方式与多Master多Slave模式-异步复制相似，修改Master配置项 brokerRole=SYNC_MASTER 即可。 