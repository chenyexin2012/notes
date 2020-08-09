## centos安装fastdfs

[libfastcommon文件下载位置](https://github.com/happyfish100/libfastcommon/releases)
[fastdfs文件下载位置](https://github.com/happyfish100/fastdfs/releases)
[fastdfs-nginx-module文件下载位置](https://github.com/happyfish100/fastdfs-nginx-module/releases)


1. 安装环境准备（如需）：

        yum install -y gcc gcc-c++
        yum -y install libevent

2. 安装libfastcommon：

        # 解压
        tar -zxvf libfastcommon-1.0.7.tar.gz
        # 运行脚本编译安装
        ./make.sh
        ./make.sh install

        # 安装完成后会在 /usr/lib64 目录下生成  libfastcommon.so 库文件，手动拷贝至 /usr/lib 目录下
        cp /usr/lib64/libfastcommon.so /usr/lib/

3. 安装fastdfs：

        # 解压
        tar -zxvf fastdfs-5.05.tar.gz
        # 运行脚本编译安装
        ./make.sh
        ./make.sh install

安装成功后会在 /usr/bin/ 下生成一些命令：

        [root@peer1 fdfs]# ls /usr/bin/ | grep fdfs*
        fdfs_appender_test
        fdfs_appender_test1
        fdfs_append_file
        fdfs_crc32
        fdfs_delete_file
        fdfs_download_file
        fdfs_file_info
        fdfs_monitor
        fdfs_storaged
        fdfs_test
        fdfs_test1
        fdfs_trackerd
        fdfs_upload_appender
        fdfs_upload_file

也会在 /usr/include/ 目录下生成两目录，包含一些头文件：

        [root@vip include]# cd /usr/include/
        [root@vip include]# ll | grep fast
        drwxr-xr-x.  2 root root   4096 Aug  9 15:45 fastcommon
        drwxr-xr-x.  2 root root   4096 Aug  9 15:55 fastdfs


同时也会在 /etc/fdfs/ 目录下面生成三个配置文件：

        [root@peer1 nginx-1.9.15]# cd /etc/fdfs/
        [root@peer1 fdfs]# ll
        total 20
        -rw-r--r--. 1 root root 1461 Aug  9 15:55 client.conf.sample
        -rw-r--r--. 1 root root 7829 Aug  9 15:55 storage.conf.sample
        -rw-r--r--. 1 root root 7102 Aug  9 15:55 tracker.conf.sample

重新命名配置文件：

        [root@peer1 fdfs]# mv client.conf.sample client.conf
        [root@peer1 fdfs]# mv storage.conf.sample storage.conf
        [root@peer1 fdfs]# mv tracker.conf.sample tracker.conf

对 tracker.conf 部分主要配置内容可进行适当修改：

        # bind an address of this host
        # empty for bind all addresses of this host
        bind_addr=

        # the tracker server port
        port=22122

        # the base path to store data and log files
        base_path=/home/yuqing/fastdfs

        # HTTP port on this tracker server
        http.server_port=8080

创建配置中的指定目录，使用以下命令启动tracker。

        /usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf start

        /usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart
        /usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf stop

4. 配置并启动storage

对 storage.conf 部分主要配置内容可进行适当修改：
        
        # the name of the group this storage server belongs to
        #
        # comment or remove this item for fetching from tracker server,
        # in this case, use_storage_id must set to true in tracker.conf,
        # and storage_ids.conf must be configed correctly.
        group_name=group1

        # the base path to store data and log files
        base_path=/home/yuqing/fastdfs

        # store存放文件的位置(store_path)，如果有多个挂载磁盘则定义多个store_path
        # store_path#, based 0, if store_path0 not exists, it's value is base_path
        # the paths must be exist
        store_path0=/home/yuqing/fastdfs
        #store_path1=/home/yuqing/fastdfs1
        #store_path2=/home/yuqing/fastdfs2

        # tracker服务端的IP和端口，可以配置多个
        # tracker_server can ocur more than once, and tracker_server format is
        #  "host:port", host can be hostname or ip address
        tracker_server=192.168.209.121:22122
        #tracker_server=192.168.209.122:22122

        # HTTP端口
        # the port of the web server on this storage server
        http.server_port=8888

创建配置中的指定目录，使用以下命令启动storage。

        /usr/bin/fdfs_storaged /etc/fdfs/storage.conf start

        /usr/bin/fdfs_storaged /etc/fdfs/storage.conf restart
        /usr/bin/fdfs_storaged /etc/fdfs/storage.conf stop

5. 使用fastDFS自带测试工具进行测试

对 client.conf 部分配置内容进行相应的修改：

        # the base path to store log files
        base_path=/home/fastdfs

        # tracker服务端的IP和端口，可以配置多个
        # tracker_server can ocur more than once, and tracker_server format is
        #  "host:port", host can be hostname or ip address
        tracker_server=192.168.31.11:22122

选择一个文件进程测试：

        [root@peer1 opt]# /usr/bin/fdfs_test /etc/fdfs/client.conf upload test.jpg
        This is FastDFS client test program v5.05

        Copyright (C) 2008, Happy Fish / YuQing

        FastDFS may be copied only under the terms of the GNU General
        Public License V3, which may be found in the FastDFS source kit.
        Please visit the FastDFS Home Page http://www.csource.org/ 
        for more detail.

        [2020-08-09 16:27:34] DEBUG - base_path=/home/fastdfs, connect_timeout=30, network_timeout=60, tracker_server_count=1, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=0, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0

        tracker_query_storage_store_list_without_group: 
                server 1. group_name=, ip_addr=192.168.31.11, port=23000

        group_name=group1, ip_addr=192.168.31.11, port=23000
        storage_upload_by_filename
        group_name=group1, remote_filename=M00/00/00/wKgfC18vs3aATQtfAAF9JumPvag836.jpg
        source ip address: 192.168.31.11
        file timestamp=2020-08-09 16:27:34
        file size=97574
        file crc32=3918511528
        example file url: http://192.168.31.11/group1/M00/00/00/wKgfC18vs3aATQtfAAF9JumPvag836.jpg
        storage_upload_slave_by_filename
        group_name=group1, remote_filename=M00/00/00/wKgfC18vs3aATQtfAAF9JumPvag836_big.jpg
        source ip address: 192.168.31.11
        file timestamp=2020-08-09 16:27:34
        file size=97574
        file crc32=3918511528
        example file url: http://192.168.31.11/group1/M00/00/00/wKgfC18vs3aATQtfAAF9JumPvag836_big.jpg

可以看到文件的路径和访问URL，则表示上传成功，但是由于此时还没有与nginx整合，因此无法使用http进行文件的下载。

6. fastdfs整合nginx：

下载并解压整合模块：

        # 其它版本可能出现与nginx不兼容的问题
        wget https://github.com/happyfish100/fastdfs-nginx-module/archive/5e5f3566bbfa57418b5506aaefbe107a42c9fcb1.zip
        unzip 5e5f3566bbfa57418b5506aaefbe107a42c9fcb1.zip
        # 重新命名
        mv fastdfs-nginx-module-5e5f3566bbfa57418b5506aaefbe107a42c9fcb1/ fastdfs-nginx-module-test

进入解压目录，接着进入 /src 目录下，修改 config 文件内容，将 /usr/local/include 修改为 /usr/include/fastdfs /usr/include/fastcommon，这对应着上面 fastdfs 生成的文件目录：

        ngx_addon_name=ngx_http_fastdfs_module

        if test -n "${ngx_module_link}"; then
        ngx_module_type=HTTP
        ngx_module_name=$ngx_addon_name
        ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon"
        ngx_module_libs="-lfastcommon -lfdfsclient"
        ngx_module_srcs="$ngx_addon_dir/ngx_http_fastdfs_module.c"
        ngx_module_deps=
        CFLAGS="$CFLAGS -D_FILE_OFFSET_BITS=64 -DFDFS_OUTPUT_CHUNK_SIZE='256*1024' -DFDFS_MOD_CONF_FILENAME='\"/etc/fdfs/mod_fastdfs.conf\"'"
        . auto/module
        else
        HTTP_MODULES="$HTTP_MODULES ngx_http_fastdfs_module"
        NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_fastdfs_module.c"
        CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon"
        CORE_LIBS="$CORE_LIBS -lfastcommon -lfdfsclient"
        CFLAGS="$CFLAGS -D_FILE_OFFSET_BITS=64 -DFDFS_OUTPUT_CHUNK_SIZE='256*1024' -DFDFS_MOD_CONF_FILENAME='\"/etc/fdfs/mod_fastdfs.conf\"'"
        fi


将配置文件拷贝至目录 /etc/fdfs/

        cp mod_fastdfs.conf /etc/fdfs/

对配置文件的部分内容进行应的修改：

        # the base path to store log files
        base_path=/tmp

        # FastDFS tracker_server can ocur more than once, and tracker_server format is
        #  "host:port", host can be hostname or ip address
        # valid only when load_fdfs_parameters_from_tracker is true
        tracker_server=tracker:22122
        #tracker_server=tracker1:22122

        # url中是否包含group名称
        # if the url / uri including the group name
        # set to false when uri like /M00/00/00/xxx
        # set to true when uri like ${group_name}/M00/00/00/xxx, such as group1/M00/xxx
        # default value is false
        url_have_group_name = true

        # store_path#, based 0, if store_path0 not exists, it's value is base_path
        # the paths must be exist
        # must same as storage.conf
        store_path0=/home/yuqing/fastdfs
        #store_path1=/home/yuqing/fastdfs1

将 libfdfsclient.so 拷贝至 /usr/lib 目录下：

        cp ngx_http_fastdfs_module.c /usr/lib/

进入fastdfs解压目录下的 conf 目录中，将http.conf mime.types文件拷贝至 /etc/fdfs/：

        cp http.conf mime.types /etc/fdfs/

安装nginx：

[nginx安装过程参考](../nginx/centos安装nginx.md)

对nginx进行编译时需加入 fastdfs-nginx-module 模块：

        ./configure --prefix=/usr/local/nginx --add-module=/usr/local/src/fastdfs-nginx-module-test/src
        make & make install

配置nginx.conf:

        # 在指定 server 内配置
        location /group1/M00/ {
            root /home/fastdfs/storage/data;
            ngx_fastdfs_moudle;
        }



[参考博客](https://www.cnblogs.com/zengpeng/p/11159611.html)
[fastdfs-nginx-module模块编译错误解决方法参考](https://codegitz.github.io/2019/01/09/fdfs%E5%AE%89%E8%A3%85Nginx%E6%A8%A1%E5%9D%97%E5%87%BA%E7%8E%B0%E9%97%AE%E9%A2%98/)