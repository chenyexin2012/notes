## centos下安装redis

    wget http://download.redis.io/releases/redis-5.0.7.tar.gz

    解压，进入解压目录， 运行 make && make PREFIX=/home/redis/services/redis-5.0.7 install 进行编译安装（PREFIX为安装路径）

    启动命令：./redis-server redis.conf
    关闭命令：./redis-cli shutdown

    开放6379端口：firewall-cmd --zone=public --add-port=6379/tcp --permanent
    重启：firewall-cmd --reload

    

