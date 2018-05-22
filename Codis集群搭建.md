# Codis集群搭建


---

Codis 是一个分布式 Redis 解决方案, 对于上层的应用来说, 连接到 Codis Proxy 和连接原生的 Redis Server 没有明显的区别 (不支持的命令列表), 上层应用可以像使用单机的 Redis 一样使用, Codis 底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务
部署环境介绍
三台节点，各自安装zk,各自起sentinel，各自起proxy，一台担任dashboard，并且交叉组成集群组建三套codis_proxy_group，并且在原有codis集群上新加项目

##安装配置zookeeper

    官方下载站点
    http://zookeeper.apache.org/releases.html#download
　　

**安装**

    tar zxf zookeeper-3.4.9.tar.gz
    mv /opt/elk/zookeeper-3.4.9 /usr/local/zookeeper
     
    echo "
    172.31.18.66 zk1
    172.31.20.206 zk2
    172.31.23.183 zk3" >> /etc/hosts
     
    mkdir /data/zookeeper                #创建zk数据存放目录
    echo 1 > /data/zookeeper/myid
    echo 2 > /data/zookeeper/myid
    echo 3 > /data/zookeeper/myid   # 创建myid文件，三台机器上不可相同       
　　

**配置**

    cd /usr/local/zookeeper/conf
    cp zoo_sample.cfg zoo.cfg
     
    vim zoo.cfg
        tickTime=2000
        initLimit=10
        syncLimit=5
        dataDir=/home/zk_data
        clientPort=2181
        server.1=172.31.18.66:2888:3888
        server.2=172.31.20.206:2888:3888
        server.3=172.31.23.183:2888:3888
    Zookeeper默认会将控制台信息输出到启动路径下的zookeeper.out中
    显然在生产环境中我们不能允许Zookeeper这样做，通过如下方法，可以让Zookeeper输出按尺寸切分的日志文件：


    zookeeper.root.logger=INFO, CONSOLE改为zookeeper.root.logger=INFO, ROLLINGFILE　　　　　　#修改conf/log4j.properties
    
    ZOO_LOG4J_PROP="INFO,CONSOLE"改为ZOO_LOG4J_PROP="INFO,ROLLINGFILE"　　　　　　　　　　　　　#修改bin/zkEnv.sh文件
　　

**zk启动测试**　　

    /usr/local/zookeeper/bin/zkServer.sh start  #启动zk
    /usr/local/zookeeper/bin/zkServer.sh status  #查看zk启动状态  正常三台节点，应有两台follower和一台leader
　　

##安装配置codis
**codis安装**

**安装GO环境**

    cd /usr/lib/golang/src/github.com/CodisLabs/codis
    go get github.com/wandoulabs/codis
    go get -u github.com/tools/godep

**下载安装codis**

    官网https://github.com/CodisLabs/codis，这里使用go安装
    cd /usr/lib/golang/src/github.com/CodisLabs/codis
    go get github.com/wandoulabs/codis
    go get -u github.com/tools/godep
    
    切换目录编译
    cd $GOPATH/src/github.com/CodisLabs/codis/
    make
    make gotest各个节点codis配置
    
    cd /data/apps/codis/src/github.com/CodisLabs/codis/config
    
    vim dashboard.toml　　　　　　　#dashboard只需要集群中的一个节点启动就可以
        coordinator_addr = "172.31.18.66:2181,172.31.20.206:2181,172.31.23.183:2181"　　#zk地址
        product_name = "us-sn-bs"　　　　　　#项目名称。此名称将会显示在dashboard的页签上　　 
        product_auth = "57b0a738d60c1d81940ec3eff7b51d4b"　　　#proxy连接dashboard的认证密码
        admin_addr = "0.0.0.0:18080"                                 #codis-adim 地址
        
    vim proxy.toml
        product_name = "us-sn-bs"　　　　#项目名称。此名称将会显示在dashboard的页签上，与dashboard配置保持一样

    vim ../admin/codis-fe-admin.sh
        COORDINATOR_ADDR="172.31.18.66:2181,172.31.20.206:2181,172.31.23.183:2181"      #zk地址
    
    vim ../admin/codis-proxy-admin.sh　　
        CODIS_DASHBOARD_ADDR="172.31.18.66:18080"                       #dashboard地址和端口
**启动codis**　　

    cd /data/apps/codis/src/github.com/CodisLabs/codis/bin　
    
**各个节点编辑redis配置文件生成脚本**

    #!/bin/env bash
     
    source /etc/init.d/functions
    help()
    {
    cat << EOF
    $0 MEM POTRS
    example $0 2G "6379 6380"
    EOF
    exit 3
    }
     
    if [ $# -ne 2 ];then
            help
    else
     
     
    CONFDIR='/data/apps/redis/conf'
    DATADIR='/data/log/redis'
    PIDDIR='/data/apps/redis/run'
    IP=`ip a |grep eth0 |grep -oP "((\d){1,3}\.){3}\d+"|grep -v \.255$`
    MEM=$1
    PORTS=$2
     
    if [ -d $CONFDIR ];then
            echo "$CONFDIR already exist"
    else
            echo "mkdir $CONFDIR"
            mkdir -p $CONFDIR
    fi
     
    if [ -d $DATADIR ];then
            echo "$DATADIR already exist"
    else
            mkdir -p $DATADIR
            echo "mkdir $DATADIR"
    fi
     
    if [ -d $PIDDIR ];then
            echo "$PIDDIR already exist"
    else
            mkdir -p $PIDDIR
            echo "mkdir $PIDDIR"
    fi
     
    chown nobody. -R $CONFDIR
    chown nobody. -R $DATADIR
    chown nobody. -R $PIDDIR
     
    for i in $PORTS;do
     
    if  [[ -f ${CONFDIR}/redis${i}.conf ]];then
            echo  "${CONFDIR}/redis${i}.conf already exist"
    else
     
    echo "daemonize yes
    pidfile $PIDDIR/redis${i}.pid
    port $i
    tcp-backlog 511
    bind ${IP}
    timeout 0
    tcp-keepalive 60
    loglevel debug
    requirepass cSjXC6DPgeX7Cp4xjx6Wl841c
    masterauth cSjXC6DPgeX7Cp4xjx6Wl841c
    logfile ${DATADIR}/redis${i}.log
    syslog-enabled no
    databases 16
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
    dir /data/db/redis${i}/
    slave-serve-stale-data yes
    repl-diskless-sync no
    repl-diskless-sync-delay 5
    repl-disable-tcp-nodelay no
    slave-priority 100
    maxclients 10000
    maxmemory ${MEM}
    maxmemory-policy allkeys-lru
    maxmemory-samples 3
    appendonly no
    appendfilename \"appendonly.aof\"
    appendfsync everysec
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    aof-load-truncated yes
    lua-time-limit 0
    slowlog-log-slower-than 10000
    slowlog-max-len 128
    latency-monitor-threshold 0
    notify-keyspace-events \"\"
    hash-max-ziplist-entries 512
    hash-max-ziplist-value 64
    list-max-ziplist-entries 512
    list-max-ziplist-value 64
    set-max-intset-entries 512
    zset-max-ziplist-entries 128
    zset-max-ziplist-value 64
    hll-sparse-max-bytes 3000
    activerehashing yes
    client-output-buffer-limit normal 0 0 0
    client-output-buffer-limit slave 256mb 64mb 60
    client-output-buffer-limit pubsub 32mb 8mb 60
    aof-rewrite-incremental-fsync yes" > ${CONFDIR}/redis${i}.conf
            [ ! -d /data/db/redis${i}/ ] && mkdir -p /data/db/redis${i}/ && chown -R nobody. /data/db/redis${i}/
            echo  "redis${i} init ok"
    fi
     
    done
    fi　
 

**各个节点生成redis配置文件**

    sh credis.sh 2G "9001 9002 9003"
　　

**各个节点启动redis**


    ./codis-server /data/apps/redis/conf/redis9001.conf
    ./codis-server /data/apps/redis/conf/redis9002.conf
    ./codis-server /data/apps/redis/conf/redis9003.conf
 

**1个节点启动dashboard，3个节点都启动proxy**　　
 

**dashboard节点，启动和安装nginx,配置web页面**

    ./codis-dashboard-admin.sh start
    ./codis-fe-admin.sh start
    ./codis-proxy-admin.sh start
    
    yum -y install nginx
    
    echo 'codisadmin:359OFx28EZ3Rs' > /etc/nginx/passwd.db
    cat>/etc/nginx/conf.d/codis.conf<EOF
    upstream codisfe-1 {
    server 127.0.0.1:9090;
    }
    server {
    listen 80;
    server_name codis-us-bs-sn-fe.mutantbox.online;
    access_log /data/log/nginx/codis-us-bs-sn-fe.mutantbox.online_access.log;
    error_log /data/log/nginx/codis-us-bs-sn-fe.mutantbox.online_error.log;
    location / {
    auth_basic "secret";
    auth_basic_user_file passwd.db;
    #include access_ctrl.conf;
    proxy_pass http://codisfe-1;
    proxy_connect_timeout 600;
    proxy_read_timeout 600;
    proxy_send_timeout 600;
    }
    }
    server {
    listen 443;
    server_name codis-us-bs-sn-fe.mutantbox.online;
    access_log /data/log/nginx/codis-us-bs-sn-fe.mutantbox.online_access.log;
    error_log /data/log/nginx/codis-us-bs-sn-fe.mutantbox.online_error.log;
    ssl on;
    ssl_certificate /etc/nginx/ssl/mutantbox.com.crt;
    ssl_certificate_key /etc/nginx/ssl/mutantbox.com.key;
    ssl_session_timeout 5m;
    ssl_protocols SSLv2 SSLv3 TLSv1;
    ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
    ssl_prefer_server_ciphers on;
    location / {
    #include access_ctrl.conf;
    auth_basic "secret";
    auth_basic_user_file passwd.db;
    proxy_pass http://codisfe-1;
    proxy_connect_timeout 600;
    proxy_read_timeout 600;
    proxy_send_timeout 600;
    }
    }
    EOF
    
    /etc/init.d/nginx restart

**两个节点启动proxy**　　

./codis-proxy-admin.sh start

**各个节点配置sentinel哨兵**

    vim /data/apps/codis/src/github.com/CodisLabs/codis/config/sentinel.conf<br>节点1
    port 26380
        #哨兵监听地址，三台节点保持一样
    dir "/tmp"
        #工作目录
    protected-mode no
        ##关闭保护模式
    daemonize yes  
        #以后台进程模式运行
    logfile "/var/log/sentinel_26380.log"
        #日志路径
     
    sentinel monitor prod_turkey_clothes_forever1 172.31.16.185 9005 2 
        # redis-master是主数据的别名，考虑到故障恢复后主数据库的地址和端口号会发生变化，哨兵提供了命令可以通过别名获取主数据库的地址和端口号
        # 172.31.16.185 9005 为初次配置时主数据库的地址和端口号，当主数据库发生变化时，哨兵会自动更新这个配置，不需要我们去关心
        # 2 该参数用来表示执行故障恢复操作前至少需要几个哨兵节点同意，一般设置为N/2+1(N为哨兵总数)
     
    sentinel auth-pass prod_turkey_clothes_forever1 cSjXC6DPgeX7Cp4xjx6Wl841c
    sentinel down-after-milliseconds prod_turkey_clothes_forever1 30000
        # 如果master在多少秒内无反应哨兵会开始进行master-slave间的切换，使用“选举”机制
    sentinel notification-script prod_turkey_clothes_forever1 /data/apps/codis/src/github.com/CodisLabs/codis/scripts/notify.sh
    sentinel parallel-syncs prod_turkey_clothes_forever1 1
    sentinel failover-timeout prod_turkey_clothes_forever1 180000
        # 如果在多少秒内没有把宕掉的那台master恢复，那哨兵认为这是一次真正的宕机，而排除该宕掉的master作为节点选取时可用的node然后等待一定的设定值的毫秒数后再来探测该节点是否恢复，如果恢复就把它作为一台slave加入哨兵监测节点群并在下一次切换时为他分配一个“选取号”。<br><br>
    节点2
    port 26380
    dir "/tmp"
    protected-mode no
    daemonize yes
    logfile "/var/log/sentinel_26380.log"
    
    sentinel monitor prod_turkey_clothes_forever2 172.31.24.102 9006 2
    sentinel auth-pass prod_turkey_clothes_forever2 cSjXC6DPgeX7Cp4xjx6Wl841c
    sentinel down-after-milliseconds prod_turkey_clothes_forever2 30000
    sentinel notification-script prod_turkey_clothes_forever2 /data/apps/codis/src/github.com/CodisLabs/codis/scripts/notify.sh
    sentinel parallel-syncs prod_turkey_clothes_forever2 1
    sentinel failover-timeout prod_turkey_clothes_forever2 180000
    
     
    
    节点3
    port 26380
    dir "/tmp"
    protected-mode no
    daemonize yes
    logfile "/var/log/sentinel_26380.log"
    
    sentinel monitor prod_turkey_clothes_forever3 172.31.17.83 9007 2
    sentinel auth-pass prod_turkey_clothes_forever3 cSjXC6DPgeX7Cp4xjx6Wl841c
    sentinel down-after-milliseconds prod_turkey_clothes_forever3 30000
    sentinel notification-script prod_turkey_clothes_forever3 /data/apps/codis/src/github.com/CodisLabs/codis/scripts/notify.sh
    sentinel parallel-syncs prod_turkey_clothes_forever3 1
    sentinel failover-timeout prod_turkey_clothes_forever3 180000

　　

sentine报警通知脚本
    
    #!/usr/bin/env bash
    ### 微信通知脚本 for sentinel ####
    ### 需要两个参数 $1 事件类型 ####
    ### 参数 $2 详细内容 ####
    ##全局参数###
    #toslist='zhanghl@mutantbox.com,chengjg@mutantbox.com,wangdq@mutantbox.com'
    toslist='18521032381,18818061825,17621390526'
    apiaddress="http://sendms_webchat.mutantbox.online/webchat/sendmessage.php"
    subject="${1}"
    msg="${2}"
    msg="[${subject}] [${msg}]"
    curl "${apiaddress}?type=sms" -d "tos=${toslist}&subject=${subject}&content=${msg}" && echo "send message [OK]" $subject"------" $msg >> /var/log/inodify_sentine.log || echo "send message [Error]" $subject"------" $msg >> /var/log/inodify_sentine.log
　　

 

**各个节点启动sentinel**

**./codis-server /data/apps/codis/src/github.com/CodisLabs/codis/config/sentinel.conf --sentinel**
　　

##至此服务器端配置完成，接下来进行web配置

    解析nginx配置的域名codis-us-bs-sn-fe.mutantbox.online
    
    页面账号密码：codisadmin : y2715jyHefkWHhr
    
    点选project
    
    一般情况下proxy会自动识别
    
     添加组和redis服务
    
    块分配给每个组
    
    添加哨兵

**至此一个集群的单project配置完毕，接下来看看如何在现有的集群中添加多个project，启动多个dashboard和proxy，以及多哨兵配置。其实很简单 复制了配置文件稍微做下修改就可以了**

**在已有的codis集群环境中新增一个project**

**配置步骤:**
 
    新增配置文件
    修改启动脚本
    启动新的集群服务
    初始化配置新的集群
    新增配置文件:
    codis 目录（/data/apps/codis/src/github.com/CodisLabs/codis)
    目录结构说明
    ├── admin（启动脚本）
    ├── bin （codis启动文件）
    │   └── assets
    ├── cmd
    │   ├── admin
    │   ├── dashboard
    │   ├── fe
    │   └── proxy
    ├── config （配置文件目录）
　　

　　

**1新建新的配置目录**

    cd /data/apps/codis/src/github.com/CodisLabs/codis
    cp -ar config config_session
　　

**2修改配置文件**

    cd config_session
    vim dashboard.toml
    dashboard.toml修改
    修改product_name
    修改product_auth
    修改admin_addr 端口
    
    ##################################################
    #                                                #
    #                  Codis-Dashboard               #
    #                                                #
    ##################################################
    # Set Coordinator, only accept "zookeeper" & "etcd" & "filesystem".
    # Quick Start
    #coordinator_name = "filesystem"
    #coordinator_addr = "/tmp/codis"
    coordinator_name = "zookeeper"
    coordinator_addr = "172.31.4.175:2181,172.31.1.18:2181,172.31.7.187:2181"
    # Set Codis Product Name/Auth.
    product_name = "pro_pt_cf"
    product_auth = ""
    # Set bind address for admin(rpc), tcp only.
    admin_addr = "0.0.0.0:18080"
    # Set arguments for data migration (only accept 'sync' & 'semi-async').
    migration_method = "semi-async"
    migration_parallel_slots = 100
    migration_async_maxbulks = 200
    migration_async_maxbytes = "32mb"
    migration_async_numkeys = 500
    migration_timeout = "30s"
    # Set configs for redis sentinel.
    sentinel_quorum = 2
    sentinel_parallel_syncs = 1
    sentinel_down_after = "30s"
    sentinel_failover_timeout = "5m"
    sentinel_notification_script = ""
    sentinel_client_reconfig_script = ""
 
    proxy.toml修改
    修改项目
    product_name（项目名字和dashboard.toml保持一致）
    product_auth （redis-server验证密码）
    session_auth = (redis-server验证密码）
    session_auth （proxy验证密码）
    admin_addr （管理端口）
    proxy_addr （proxy端口）
    ##################################################
    #                                                #
    #                  Codis-Proxy                   #
    #                                                #
    ##################################################
    # Set Codis Product Name/Auth.
    product_name = "pro_pt_php_session"
    product_auth = ""
    # Set auth for client session
    #   1. product_auth is used for auth validation among codis-dashboard,
    #      codis-proxy and codis-server.
    #   2. session_auth is different from product_auth, it requires clients
    #      to issue AUTH <PASSWORD> before processing any other commands.
    session_auth = ""
    # Set bind address for admin(rpc), tcp only.
    admin_addr = "0.0.0.0:11081"
    # Set bind address for proxy, proto_type can be "tcp", "tcp4", "tcp6", "unix" or "unixpacket".
    proto_type = "tcp4"
    proxy_addr = "0.0.0.0:19010"
　　

**3修改启动脚本**

    复制原有脚本
    cd /data/apps/codis/src/github.com/CodisLabs/codis/admin
    cp codis-dashboard-admin.sh codis-dashboard-admin-new.sh
    cp codis-proxy-admin.sh codis-proxy-admin-new.sh
 
    修改脚本
    codis-dashboard-admin-new.sh
     
    CODIS_CONF_DIR=$CODIS_ADMIN_DIR/../config_session (配置为新的配置文件目录)
    CODIS_DASHBOARD_PID_FILE（修改PID文件）
    CODIS_DASHBOARD_LOG_FILE（修改日志文件）
    CODIS_DASHBOARD_DAEMON_FILE （修改）
    codis-proxy-admin-new.sh
     
    CODIS_CONF_DIR=$CODIS_ADMIN_DIR/../config_session(配置文件)
    CODIS_PROXY_PID_FILE（PID文件）
    CODIS_PROXY_LOG_FILE
    CODIS_PROXY_DAEMON_FILE
    CODIS_DASHBOARD_ADDR (管理端口，对应dashboard配置文件中的端口)
　

**4启动新的project**

    启动顺序
    Dashboard
    Porxy
    Codis-server
    Sentinel　　


**5配置哨兵**
    配置哨兵不做相信赘述，只是把原来的配置文件复制过来修改下端口，prodc_name 和地址就可以了，同样的也是每个节点都需要配置。

