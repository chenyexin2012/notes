## centos安装RabbitMQ

1. 安装Erlang环境，RabbitMQ是Erlang语言编写的，RabbitMQ版本必须和Erlang保持一致

[版本对照](https://www.rabbitmq.com/which-erlang.html)

下载路径：http://erlang.org/download/otp_src_21.3.tar.gz

依赖包安装（如需）：

    yum -y install ncurses-devel
    yum search libtool
    yum search libtool-ltdl-devel
    yum install libtool
    yum install libtool-ltdl-devel
    yum install gcc-c++
    yum install erlang-doc
    yum install erlang-jinterface

解压下载文件，并在 /usr/local/ 下创建文件夹 erlang

进入解压目录，运行命令 ./configure --prefix=/usr/local/erlang 指定安装目录

运行命令 make & make install 编译安装

添加环境变量并刷新

        echo 'export PATH=$PATH:/usr/local/erlang/bin' >> /etc/profile
        source  /etc/profile

运行 erl 看能否进入命令行

2. 安装RabbitMQ

安装依赖：

        yum install socat

下载rpm包：https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.4/rabbitmq-server-3.8.4-1.el7.noarch.rpm



