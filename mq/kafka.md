## centos安装kafka

    wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.4.0/kafka_2.13-2.4.0.tgz

    解压文件，进入conf文件夹，修改配置文件server.properties，部分配置说明如下：

    port：kafka对外提供服务的端口，默认为9092
    broker.id：默认为0，每一台服务器的broker.id不能相同。
    zookeeper.connect：zookeeper服务地址，如localhost:2181，如有多台服务，可以全部加上并以逗号分隔。
    listeners：在配置集群的时候，必须设置，不然以后的操作会报找不到leader的错误。

    进入bin文件夹，运行命令 ./kafka-server-start.sh -daemon ../config/server.properties 启动服务。

    解决安装后无法远程访问的问题：
    1. 检查防火墙
    2. 修改配置文件server.properties（listeners=PLAINTEXT://:9092，PLAINTEXT://[本机IP]:9092）

    常用命令：
    # 创建topic
    kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
    # 查看所有topic
    kafka-topics.sh --list --bootstrap-server localhost:9092
    # 生产消息（Ctrl + C 退出）
    kafka-console-producer.sh --broker-list localhost:9092 --topic test
    # 消费消息（Ctrl + C 退出）
    kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
    
