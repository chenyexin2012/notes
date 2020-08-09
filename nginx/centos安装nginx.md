## centos安装nginx

[nginx安装包下载地址](http://nginx.org/download/)

1. 安装环境准备（如需）

        # 编译环境
        yum -y install gcc
        # nginx的http模块使用pcre来解析正则表达式
        yum install -y pcre pcre-devel
        # nginx使用zlib对http包的内容进行gzip
        yum install -y zlib zlib-devel
        # 安全通信的基石
        yum install -y openssl openssl-devel

2. 下载nginx安装包至指定目录，解压

        # 以 nginx-1.9.15 为例
        wget http://nginx.org/download/nginx-1.19.0.tar.gz
        # 解压文件
        tar -zxvf nginx-1.19.0.tar.gz

3. 进入文件目录，生成 Makefile 文件

        ./configure

默认的安装目录为 /usr/local/nginx，也可以通过 ./configure 添加参数 --prefix=/usr/local/nginx 进行指定

4. 编译安装

        make && make install

5. 进入 /usr/local/nginx/sbin 目录

        # 启动脚本
        nginx
        # 关闭
        nginx -s stop
        # 重启
        nginx -s reload

启动成功后可直接通过80端口访问默认页面：Welcome to nginx!

6. 配置文件路径为 /usr/local/nginx/conf/nginx.conf，下面为配置文件部分内容说明

        # 运行用户
        #user  nobody;
        # 启动进程数，一般可以与CPU核心数相等
        worker_processes  1;

        # 全局错误日志
        #error_log  logs/error.log;
        #error_log  logs/error.log  notice;
        #error_log  logs/error.log  info;

        # pid 文件路径
        #pid        logs/nginx.pid;

        # 配置影响nginx服务器或与用户的网络连接
        events {
                # 设置网路连接序列化，防止惊群现象发生，默认为on
                #accept_mutex on;   
                # 设置一个进程是否同时接受多个网络连接，默认为off
                #multi_accept on;  
                # 事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport
                #use epoll;      
                #最大连接数，默认为512
                worker_connections  1024;    
        }

        # HTTP 模块配置
        http {
                # 设定mime类型,类型由mime.type文件定义
                include       mime.types;
                # 默认文件类型，默认为text/plain
                default_type  application/octet-stream;

                # 自定义日志格式
                #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                #                  '$status $body_bytes_sent "$http_referer" '
                #                  '"$http_user_agent" "$http_x_forwarded_for"';
                
                # 日志路径
                #access_log  logs/access.log  main;

                # 指定 nginx 是否调用 sendfile 函数（zero copy 方式）来输出文件，对于普通应用，必须设为 on，如果用来进行下载等应用
                # 磁盘IO重负载应用，可设置为 off，以平衡磁盘与网络I/O处理速度，降低系统的uptime。
                sendfile        on;
                #tcp_nopush     on;

                # 连接超时时间
                #keepalive_timeout  0;
                keepalive_timeout  65;

                # 开启gzip压缩
                #gzip  on;

                # 配置负载均衡的服务器列表
                upstream myserver {
                        # weigth参数表示权值，权值越高被分配到的几率越大
                        server 192.168.31.11:9090  weight=1;
                        server 192.168.31.13:9090  weight=4;
                }

                # 具体服务配置，可以配多个
                server {
                        # 监听端口
                        listen       80;
                        # 监听地址
                        server_name  localhost;

                        #charset koi8-r;

                        #access_log  logs/host.access.log  main;

                        # 默认请求
                        location / {
                                # 服务器的默认网站根目录位置
                                root   html;
                                # 首页索引文件的名称
                                index  index.html index.htm;
                        }

                        # 定义错误提示页面
                        #error_page  404              /404.html;
                        # redirect server error pages to the static page /50x.html
                        #
                        error_page   500 502 503 504  /50x.html;
                        location = /50x.html {
                                root   html;
                        }

                        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
                        #
                        #location ~ \.php$ {
                        #    proxy_pass   http://127.0.0.1;
                        #}

                        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
                        #
                        #location ~ \.php$ {
                        #    root           html;
                        #    fastcgi_pass   127.0.0.1:9000;
                        #    fastcgi_index  index.php;
                        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
                        #    include        fastcgi_params;
                        #}

                        # 配置禁止访问文件
                        # deny access to .htaccess files, if Apache's document root
                        # concurs with nginx's one
                        #
                        #location ~ /\.ht {
                        #    deny  all;
                        #}
                }


                # another virtual host using mix of IP-, name-, and port-based configuration
                #
                #server {
                #    listen       8000;
                #    listen       somename:8080;
                #    server_name  somename  alias  another.alias;

                #    location / {
                #        root   html;
                #        index  index.html index.htm;
                #    }
                #}


                # HTTPS server
                #
                #server {
                #    listen       443 ssl;
                #    server_name  localhost;

                #    ssl_certificate      cert.pem;
                #    ssl_certificate_key  cert.key;

                #    ssl_session_cache    shared:SSL:1m;
                #    ssl_session_timeout  5m;

                #    ssl_ciphers  HIGH:!aNULL:!MD5;
                #    ssl_prefer_server_ciphers  on;

                #    location / {
                #        root   html;
                #        index  index.html index.htm;
                #    }
                #}

                # 配置服务
                server {
                    listen       9090;
                    server_name  localhost;
                    
                    location / {
                        # 将请求转向定义的 myserver 服务列表
                        proxy_pass  http://myserver;
                        root   html;
                        index  index.html index.htm;
                    }
                }

        }