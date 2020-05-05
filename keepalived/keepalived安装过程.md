## centos安装keepalived

1. 安装基础依赖包（如需）

    yum install gcc
    yum -y install openssl-devel
    yum -y install libnl libnl-devel
    yum -y install libnfnetlink-devel
    yum -y install net-tools

2. 官网下载安装文件

    官网地址：https://www.keepalived.org/download.html

    如下载2.0.20版本：
    wget https://www.keepalived.org/software/keepalived-2.0.20.tar.gz

3. 解压安装包，将解压文件移动至 /usr/local/keepalived 目录下

4. 生成 Makefile 文件

    ./configure

5. 编译安装

    make && make install

    完成后会在以下路径生成文件：

    /usr/local/etc/keepalived/keepalived.conf
    /usr/local/etc/sysconfig/keepalived
    /usr/local/sbin/keepalived
    /usr/lib/systemd/system/keepalived.service 

6. 在默认配置文件夹下添加配置文件

    创建文件夹：
    mkdir /etc/keepalived
    拷贝配置文件（按需修改）：
    cp /usr/local/etc/keepalived/keepalived.conf /etc/keepalived/

7. 启动关闭服务

    systemctl start keepalived
    systemctl stop keepalived

8. 添加开机自启

    systemctl enable keepalived


9. 防火墙放过vrrp协议（如需）

    firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
    firewall-cmd --direct --permanent --add-rule ipv4 filter OUTPUT 0 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
    firewall-cmd --reload


## 