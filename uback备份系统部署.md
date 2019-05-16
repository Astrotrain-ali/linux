# uback备份系统部署

---
[备份系统githup地址](https://github.com/lustlost/ubackup)
## 组件说明
- 服务端：需要ElasticSearch,Redis，Mysql
  - elastic:备份系统的搜索引擎，可以单独安装与集群中某一台机器
  - redis:备份系统的队列，可以单独安装与集群中某一台机器
  - mysql:数据库，可以单独安装与集群中某一台机器
  - nginx:做dashboard的跳转，一般与dashboard安装同一台机器
- dashboard页面，可以单独安装在集群中某一台机器
- 备份系统源码：所有服务端都需安装

## 服务端依赖
```
pip install requests
yum install supervisor MySQL-python -y
pip install redis
```
## 客户端依赖
```
yum install python-redis -y
yum install rsync xinetd -y
sed -i '/disable/ s#yes#no#g' /etc/xinetd.d/rsync
cd ubackup/client
cp uuzuback.conf /etc/
```
```
rsync:模块格式
[backup]
path = /data/backup/
hosts allow = 0.0.0.0/0
read only = yes
```
---
## 部署环境
三台机器做为服务端集群。操作系统均为Centos6.5

-   10.53.0.188
    -   jdk,elastic(5.2.1版本),备份系统节点1，磁盘信息上报
-   10.53.0.189
    -   redis,nginx,mysql,dashiboard，备份系统节点2，磁盘信息上报，队列信息上报   
-   10.53.0.190
    -   备份系统3，磁盘信息上报

---
#服务端
### 部署步骤
-   10.53.0.188
#### jdk
```
#安装jdk
yum -y install java-1.8.0-openjdk   
```
#### elastic
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.2.1.tar.gz 
tar xzvf elasticsearch-5.2.1.tar.gz
mv elasticsearch-5.2.1 /usr/local/elasticsearch
#修改运行内存大小
vim /usr/local/elasticsearch/config/jvm.options 									
-Xms1g
-Xmx1g
#修改elastic可创建的文件描述符大小
vim /etc/security/limits.conf   													
elastic hard nofile 65536
elastic soft nofile 65536
#修改限制用户拥有的最大进程数量
vim /etc/security/limits.d/90-nproc.conf    										
*          soft    nproc     1024改为4096
#修改进程可以拥有的VMA(虚拟内存区域)的数量
vim /etc/sysctl.conf    															
vm.max_map_count=655360 #新增
sysctl -p
#新增elastic组及用户, 因为ES不允许root用户启动
groupadd elastic    																
useradd elastic -g elastic
echo 'elastic' | passwd --stdin elastic
chown -R elastic:elastic /usr/local/elasticsearch    
#修改elastic配置文件,由于是单机运行es，只需要改这些配置就ok
vim /usr/local/elasticsearch/config/elasticsearch.yml   							
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
network.host: 0.0.0.0
su elastic
#启动elasticsearch   不能使用root用户启动
/usr/local/elasticsearch/bin/elasticsearch -d 
#验证										
curl http://localhost:9200    														
```
#### 备份系统服务端
-   先安装好服务端依赖
-   安装备份源码进行配置
  
```
#下载源码
git clone https://github.com/lustlost/ubackup.git 
#初始化esmapper,此处注意修改脚本中的url地址中的索引名称，这里取用mappings，如r=requests.put('http://127.0.0.1:9200/mappings/',data=json.dumps(a))
python ubackup/server/create_es_mapper.py   
cd ubackup/server/
cp uuzuback.conf /etc/
#配置文件
vim /etc/uuzuback.conf 
[global]
work_thread = 4 #执行rsync拉取的线程数量
rsync_bwlimit = 62000 #rsync带宽限制
server_root_path = /data/redisbak/ #本地备份存放根目录
reserve = 1000 #单位GB,本机磁盘保留大小，如果小于这个值就会有错误日志输出
interval = 0 #每个线程拉取备份后，休息的秒数，可以用来控制拉取速度的
redis_host = 10.53.0.189 #redis地址
redis_port = 6381 #redis端口
redis_queue = level1 galaxy #队列名称，越前面的优先级越高
log = /var/log/uuzuback.log 
error_log = /var/log/uuzuback.error
myip = 10.53.0.188 #本机IP
retry = 3 #拉取重试次数
message_redis_host=10.53.0.189 #全局队列地址
message_redis_port=6381 #全局队列端口
message_queue=galaxy #全局队列名称
mysqlkeeptime = 15 #保留字段
rediskeeptime = 15 #保留字段
node_id=1 #节点ID，从Dashboard中获取，用于上报节点信息用
```
-   启动服务
```
vim /etc/supervisord.conf #加入配置段，server端执行文件路径根据实际情况配置
    [program:uuzu_backup_server]
    command=/usr/bin/python /usr/local/ubackup/server/uuzubackup_server.py
    autorestart=true
    autostart=true
    stdout_logfile=/var/log/uuzu_backup_server.log
    
    [program:to_es]
    command=/usr/bin/python /usr/local/ubackup/server/to_es.py
    autorestart=true
    autostart=true
    stdout_logfile=/var/log/uuzu_backup_to_es.log
vim /usr/local/ubackup/server/to_es.py  #打开to_es.py修改5-9行的redis,es,dashboard连接信息
    redis_handle = Redis(host='10.53.0.189',port=6381)
    redis_handle_ex = Redis(host='10.53.0.189',port=6381,db=9)
    message_queue='message'
    es_url='http://10.53.0.188:9200'
    dashboard_url='http://10.53.0.189:5000'
	r=requests.post("/mappings/table/"%es_url,data=json.dumps(message)) #修改此处写死的url
/etc/init.d/supervisord restart #使用supervisord启动
chkconfig supervisord on    #开机自启
```
#### 磁盘信息上报
```
在节点服务器上
修改 server/update_disk.py中的dashboard_url
添加corntab
python update_disk.py 加入crontab
配置文件中的node_id配置的ID，配置为Dashboard上的节点ID
* * * * * /usr/bin/python /usr/local/ubackup/server/update_disk.py 1
```
---

-   10.53.0.189
####redis
```
yum install redis -y    #安装redis
mkdir -pv /data/apps/redis/conf #配置文件
vim /data/apps/redis/conf/redis6381.conf
    daemonize yes
    pidfile /data/apps/redis/run/redis6381.pid
    port 6381
    tcp-backlog 511
    bind 10.53.0.189
    unixsocket /tmp/redis6381.sock
    unixsocketperm 777
    timeout 0
    tcp-keepalive 60
    loglevel debug
    logfile /data/log/redis/redis6381.log
    syslog-enabled no
    databases 16
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename liberators-redis-bak-3-10-11-113-251_6381.rdb
    dir /data/db/redis6381/
    slave-serve-stale-data yes
    repl-diskless-sync no
    repl-diskless-sync-delay 5
    repl-disable-tcp-nodelay no
    slave-priority 100
    maxclients 10000
    maxmemory 1G
    maxmemory-policy noeviction
    maxmemory-samples 3
    appendonly no
    appendfilename "liberators-redis-bak-3-10-11-113-251_6381.aof"
    appendfsync everysec
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    aof-load-truncated yes
    lua-time-limit 0
    slowlog-log-slower-than 10000
    slowlog-max-len 128
    latency-monitor-threshold 0
    notify-keyspace-events ""
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
    aof-rewrite-incremental-fsync yes
redis-server /data/apps/redis/conf/redis6381.conf   #启动
```
####nginx
```
yum install nginx -y
vim /etc/nginx/nginx.conf   #优化配置文件nginx.conf
    user              nobody;
    worker_processes  8;
    
    error_log  /var/log/nginx/error.log;
    pid        /var/run/nginx.pid;
    worker_rlimit_nofile 51200;
    
    events {
        use epoll;
        worker_connections  65532;
    }
    
    
    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;
    
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for""$request_time" "$upstream_response_time"';
    
       log_format  ups_log  '$http_x_forwarded_for - $remote_addr - $remote_user  [$time_local]  '
                                 ' "$request"  $status $body_bytes_sent '
                                 ' "$http_referer"  "$http_user_agent" '
                           '$upstream_addr $upstream_status $upstream_http_cookie';
    
        server_tokens off;
    
        fastcgi_intercept_errors on;
        server_names_hash_bucket_size 256;
        client_header_buffer_size 8192k;
        large_client_header_buffers 4 256k;
        client_max_body_size 30m;
        sendfile       on;
        tcp_nopush     on;
        tcp_nodelay    on;
        keepalive_timeout  120;
        send_timeout 600;
    
        gzip  on;
        gzip_http_version 1.1;
        gzip_disable     "MSIE [1-6]\.";
        gzip_proxied any;
        gzip_min_length 1k;
        gzip_comp_level 6;
        gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    
        client_body_temp_path /data/nginxtmp 3 4;
        proxy_temp_path    /data/nginx_proxy_tmp ;
        proxy_cache_path /data/nginxcgi_cache  levels=1:2  keys_zone=nginxcgi_php:10m inactive=1d max_size=100m;
        
        # Load config files from the /etc/nginx/conf.d directory
        # The default server is in conf.d/default.conf
        include /etc/nginx/conf.d/*.conf;
    
    }


vim /etc/nginx/conf.d/backup.mutantbox.com.conf #备份页面配置文件
    upstream uuzuback {
        server 127.0.0.1:5000;
    }
    
    server {
        listen 80;
        server_name backup.mutantbox.com;
        access_log  /var/log/nginx/access_backup.mutantbox.com.log;
    
        location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-NginX-Proxy true;
            proxy_pass http://uuzuback;
            proxy_connect_timeout    600;
            proxy_read_timeout       600;
            proxy_send_timeout       600;
        }
    }
    
    
    server {
        listen       443;
        server_name  backup.mutantbox.com;
        access_log  /var/log/nginx/access_443_backup.mutantbox.com.log;
        error_log /var/log/nginx/error_443_backup.mutantbox.com.log;
        ssl                  on;
        ssl_certificate      /etc/nginx/ssl/mutantbox.com.crt;
        ssl_certificate_key  /etc/nginx/ssl/mutantbox.com.key;
    
        ssl_session_timeout  5m;
    
        ssl_protocols  SSLv2 SSLv3 TLSv1;
        ssl_ciphers  ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
        ssl_prefer_server_ciphers   on;
    
        location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-NginX-Proxy true;
            proxy_pass http://uuzuback;
            proxy_connect_timeout    600;
            proxy_read_timeout       600;
            proxy_send_timeout       600;
        }
    }
```
#### Mysql
```
wget http://52.197.150.65:8889/rpm/mysql-5.6.27-1.el6.x86_64.rpm    #安装mysql
rpm -ivh mysql-5.6.27-1.el6.x86_64.rpm --nodeps --replacepkgs  --force
cd /usr/local/mysql
mkdir /data/mysql3306/log -pv &&  chown mysql. /data/mysql3306 -R 
mkdir /tmp/mysql -pv && chown -R mysql. /tmp/mysql
/usr/local/mysql/scripts/mysql_install_db --user=mysql --datadir=/data/mysql3306 --basedir=/usr/local/mysql
chkconfig mysqld on
vim /etc/my.cnf #配置文件
    [client]
    port            = 3306
    socket          = /tmp/mysql.sock
    
    [mysqld]
    user     	= mysql
    datadir         = /data/mysql3306
    port            = 3306
    socket          = /tmp/mysql.sock
    basedir 	= /usr/local/mysql
    pid-file 	= /data/mysql3306/mysql.pid
    
    general_log = OFF
    #log_output = /DB/log/general.log
    general_log_file = /data/mysql3306/log/general.log
    
    binlog_format = MIXED
    transaction-isolation = READ-COMMITTED
    expire_logs_days = 3
    log-bin=/data/mysql3306/log/mysql-bin
    log-slave-updates = 1
    relay_log=/data/mysql3306/log/relay-bin
    
    #log_warnings
    #log_long_format
    log-error=/data/mysql3306/log/error.log
    
    slow_query_log = 1
    long_query_time = 3
    slow-query-log-file=/data/mysql3306/log/slowquery.log
    tmpdir=/tmp/mysql
    
    #read_only = 1
    #slave_skip_errors = all
    #skip-networking
    #skip-external-locking
    
    
    skip_name_resolve
    event_scheduler=ON
    sql_mode = 0
    back_log = 1000
    max_connections = 10000
    max_connect_errors = 20000
    table_open_cache = 4096
    max_allowed_packet = 32M
    binlog_cache_size = 4M
    max_heap_table_size = 128M
    read_buffer_size = 8M
    read_rnd_buffer_size = 32M
    sort_buffer_size = 16M
    join_buffer_size = 16M
    thread_cache_size = 256
    #[CPU数量]*(2..4)
    thread_concurrency = 8
    
    #查询缓冲
    query_cache_size = 512M
    query_cache_limit = 4M
    query_cache_type = 1
    ft_min_word_len = 4
    #memlock
    #default-character-set=utf-8
    default-storage-engine = innodb
    thread_stack = 512K
    transaction_isolation = REPEATABLE-READ
    tmp_table_size = 512M
    explicit_defaults_for_timestamp = true
    
    log_bin_trust_function_creators = 1
    
    server_id = 1
    replicate-ignore-db = test
    replicate-ignore-db = information_schema
    gtid-mode = ON
    enforce-gtid-consistency = ON
    slave-parallel-workers = 4
    sync_binlog = 1
    
    
    #30%cache
    key_buffer_size = 64M
    bulk_insert_buffer_size = 32M
    myisam_sort_buffer_size = 32M
    myisam_max_sort_file_size = 10G
    myisam_repair_threads = 1
    myisam_recover_options = BACKUP,FORCE
    
    #skip-innodb
    innodb_use_sys_malloc = ON
    
    #%80
    innodb_buffer_pool_size = 256M
    innodb_data_file_path = ibdata1:10M:autoextend
    innodb_file_per_table = 1
    
    #该参数值之和=2*cpu个数*cpu核数;如果你的系统读>写,可以设置innodb_read_io_threads值相对大点
    #数据库写操作时的线程数，用于并发
    innodb_write_io_threads = 8
    
    #该参数值之和=2*cpu个数*cpu核数;如果你的系统读>写,可以设置innodb_read_io_threads值相对大点
    #数据库读操作时的线程数，用于并发。
    innodb_read_io_threads = 8
    
    
    innodb_thread_concurrency = 16
    innodb_flush_log_at_trx_commit = 1
    innodb_log_buffer_size = 16M
    innodb_max_dirty_pages_pct = 90
    innodb_lock_wait_timeout = 12000
    innodb_log_files_in_group = 3
    #innodb_flush_method=fdatasync
    #innodb_fast_shutdown
    #innodb_log_file_size = 512M
    #innodb_force_recovery=1
    
    [mysqldump]
    quick
    max_allowed_packet = 256M
    
    [mysql]
    no-auto-rehash
    #safe-updates
    
    [myisamchk]
    key_buffer_size = 2048M
    sort_buffer_size = 2048M
    read_buffer = 32M
    write_buffer = 32M
    
    [mysqlhotcopy]
    interactive-timeout
    
    [mysqld_safe]
    open-files-limit = 65532

/etc/init.d/mysqld restart


mysql   #用户授权
    GRANT ALL PRIVILEGES ON ubackup.* TO 'ubackup'@'127.0.0.1' IDENTIFIED BY 'edJelrZuAgpNeLyn' WITH GRANT OPTION;
    DELETE FROM mysql.user WHERE user in ('root','');
    SELECT user,host FROM mysql.user WHERE user in ('root','');
    FLUSH PRIVILEGES;

create database ubackup;    #创建数据库
```
#### dashboard
```
git clone https://github.com/pallets/flask #安装flask
cd flask
python setup.py install
git clone https://github.com/lustlost/ubackup.git #下载源码
yum install libffi-devel -y     #安装dashboard依赖
pip install flask-sqlalchemy
pip install flask_login
pip install redis
pip install flask_restless
cd ubackup/dashboard
pip install -r requirements.txt
vim /usr/local/ubackup/dashboard/myapp/views/front.py # 修改备份列表获取地址和显示信息
	b = requests.get("http://127.0.0.1:9200/mappings/table/_search", data=json.dumps(q_string))
	return json.dumps(b.json())
vim config.py   #修改config.py，配置好数据库连接串
    SECRET_KEY = 'sdbernesemountaindog'
    
    SQLALCHEMY_DATABASE_URI = 'mysql://ubackup:edJelrZuAgpNeLyn@127.0.0.1/ubackup?charset=utf8'
    SQLALCHEMY_POOL_RECYCLE = 60
    SQLALCHEMY_TRACK_MODIFICATIONS = True
ln -s /usr/local/mysql/lib/libmysqlclient.so.18 /usr/lib64/libmysqlclient.so.18 #建表语句报错需要执行此命令，原因是没有找到/usr/lib64/libmysqlclient.so.18，所有做一个软连
python -c 'from myapp import db;db.create_all()'
nohup python runserver.py  &    #运行dashoboard
```
#### 备份系统
```
参考节点1安装步骤
```
#### 队列信息上报
```
在redis服务器上
修改 server/update_queue.py中的dashboard_url
添加crontab
python update_queue.py {{ID}} 加入crontab ，ID
配置文件使用任意节点的配置,如下；
* * * * * /usr/bin/python /opt/ubackup/server/update_queue.py 2
```
#### #### 磁盘信息上报
```
在节点服务器上
修改 server/update_disk.py中的dashboard_url
添加corntab
python update_disk.py 加入crontab
配置文件中的node_id配置的ID，配置为Dashboard上的节点ID
* * * * * /usr/bin/python /usr/local/ubackup/server/update_disk.py 2
```
---
-   10.53.0.190
#### 备份系统
```
参考节点1安装步骤
```
#### #### 磁盘信息上报
```
在节点服务器上
修改 server/update_disk.py中的dashboard_url
添加corntab
python update_disk.py 加入crontab
配置文件中的node_id配置的ID，配置为Dashboard上的节点ID
* * * * * /usr/bin/python /usr/local/ubackup/server/update_disk.py 3
```
---
#客户端
###部署步骤
####clien.py
-   客户端脚本需要定时任务在每一台客户端主机上，如下所示；
    -   把python uuzuback_client.py 加入crontab，cron的间隔时间和配置文件中的interval保持一致
```
8 */1 * * * /usr/bin/python  /usr/local/uuzuback_client.py
```

-   客户端配置文件样例
```
    [global]
    server_id = 111111
    game_id = 31
    op_id = 1
    redis_host = 0.0.0.0
    redis_port = 6381
    redis_queue = message
    my_ip = 127.0.0.0.1
    log = /var/log/uuzu_back.log
    error_log = /var/log/uuzu_back.error
    
    [mysql3308]
    back_type = mysql
    instance = 3308
    rsync_model = backup/database_3308
    back_dir = /data/backup/database_3308/
    back_log = /var/log/mysql_3308.log
    last_all_log = /var/log/last_3308.log
    script = sh /usr/local/uuzuback/mysql_backup.sh /etc/my.cnf
    
    [redis6379]
    back_type = redis
    instance = 6381
    rsync_model = backup/redisbase_6381
    back_dir = /data/backup/redisbase_6381/
    back_log = /var/log/redis_6381.log
    script = sh /usr/local/uuzuback/redis_backup.sh /data/conf/redis_conf
```
-   需要确保/etc/rsyncd.conf配置文件包含以下模块（具体可以根据实际情况修改）
```
[backup]
path = /data/backup/
hosts allow = 0.0.0.0/0
read only = yes
```

####备份脚步
-   此系统只负责备份异地传输，不负责如何备份 具体备份脚本用户自行编写，只要按照要求把备份信息写入对应日志文件即可
```
备份脚本规范

1.每次都是全备的方式 （例如redis通过RDB每次都是全备）

2.全备+增量 的方式（例如Mysql通过Xtrabackup）
 
脚本日志生成规范：

    备份成功：back_log 第一行为ok，第二行为文件名

    备份失败：back_log 第一行为wrong，第二行为错误信息

如果是第2种方式，则在生成back_log的时候，同时生成一份相同的日志信息在last_all_log 日志中
```

-   现使用的redis备份脚本样例
```
#!/usr/bin/env bash

#本地备份保留时间(分钟)
MINUTES='720'
#备份文件存放目录
backup_dir='/data/backup'
#操作日志记录
run_log='/var/log/redis_backup.log'



log()
{
	log_date=`date +"%F %T"`
	echo "${log_date} $*" |tee -a  ${run_log}
}
check_maxmem()
{
	maxmem=$1
	nowmem=`free -g|awk '/Mem/ {print $4 + $NF}'`
	if [[ "${nowmem}" -gt  "${maxmem}" ]];then
		log  'mem ok'
	else
		log_date=`date +"%F %T"`
		content="${log_date} ${hostname} ERROR:nowmem is : ${nowmem} maxmem limit is :${maxmem}"
		log "$content"
		curl "http://139.196.52.9/webchat/sendmessage.php?type=email" -d "tos=${toslist}&subject=${subject}&content=${content}"
		log "wrong/nFree Mem is less than ${maxmem}"
		exit 3
	fi
}



cp_file()
{
	sdata=`date +%s`
	redis_config=$1
	rdb_file=$(awk '/^dbfilename/ {print $2}' $redis_config)
	rdb_dir=$(awk '/^dir/ {print $2}' $redis_config)
	redis_port=$(awk '/^port/ {print $2}' $redis_config)
	out=`rpm -qa pigz`
        ret=`echo $out|grep pigz`
	if [[ "x${ret}x" == 'xx' ]];then
                yum -y install pigz
	else
		log "alread installed pigz"
	fi

	if [ -d ${backup_dir}/redisbase_${redis_port} ];then
		cp -ar ${rdb_dir}${rdb_file} ${backup_dir}/redisbase_${redis_port}/${rdb_file}_${sdata} && cd ${backup_dir}/redisbase_${redis_port} && pigz -9 -p 4 ${rdb_file}_${sdata}
		echo ${rdb_file}_${sdata}.gz >>${LOGFILE}
	else
		mkdir -p ${backup_dir}/redisbase_${redis_port}
		cp -ar ${rdb_dir}${rdb_file} ${backup_dir}/redisbase_${redis_port}/${rdb_file}_${sdata} && cd ${backup_dir}/redisbase_${redis_port} && pigz -9 -p 4 ${rdb_file}_${sdata}
		echo ${rdb_file}_${sdata}.gz >>${LOGFILE}
	fi

}


redis_save()
{
	redis_config=$1
	rdb_dir=$(awk '/^dir/ {print $2}' $redis_config)
	redis_socket=$(awk '/^unixsocket / {print $2}' $redis_config)
	echo save | redis-cli -s ${redis_socket}
	findout=`find ${rdb_dir} -type f -name *.rdb -cmin -2`
	echo $findout
	if [[ "XX" == "X${findout}X" ]];then
		stat=wrong
	else
		stat=ok
	fi
	echo $stat > ${LOGFILE}
}

function del_bak()
{
        # process: del_bak function
        # Syntax:  del_bak
        # Author: Felix.li
        # Returns:
        #       N/A
        echo $1
        for dbfile in `find "${1}/" -name "*.rdb*" -type f -mmin +${MINUTES}`; do
                log "delete from ${dbfile}"
                rm -f ${dbfile}
        done
}

if [ $# -ne 1 ];then
	echo "args ERROR"
else
	#检查内存
	check_maxmem 2
	redis_config=$1
	REDISPORT=$(awk '/^port/ {print $2}' $redis_config)
	LOGFILE='/var/log/redis_'${REDISPORT}'.log'
	#备份redis实例
	redis_save ${redis_config}
	cp_file ${redis_config}
	del_bak ${backup_dir}
fi
```

-   现使用的Mysql备份脚步样例
```
    #!/bin/bash
    #
    #       Mysql Complete Hot Backup
    #       2011-02-25 14:20:08
    #       Version V1.6
    ## @modify by lust @ date 2014-6-3
    
    # Source function library.
    mysql_config_file=$1
    . /etc/init.d/functions
    
    PATH=$PATH:/data/mysql/bin
    export PATH
    
    # 保留备份的天数
    DAY=1
    
    PORT=$(awk -F"=" '$1~/\[mysqld\]/,$FNR {if ($1~/port/) {print $2;exit}}' $mysql_config_file  | xargs)
    USER="dbmanger"
    MYSQLHOST="127.0.0.1"
    PASSWORD="edJelrZuAgpNeLyn"
    
    
    
    IOPS=30
    USE_MEM="10M"
    DATE=$(date '+%F_%H-%M')
    HOUR=$(date +%H)
    BACKUP_PATH="/data/mysqlbak/database_${PORT}"
    LOG_FILE="/var/log/xtrabackup_${PORT}.log"
    MYSQL_LOG="/var/log/mysql_${PORT}.log"
    HOST=$(/sbin/ifconfig | grep -E '([0-9]+)\.([0-9]+)\.([0-9]+)\.([0-9]+)' | awk '{print $2}' | cut -d":" -f2 | grep -E "^10\.?\.")
    LAST_ALL_LOG="/var/log/last_${PORT}.log"
    
    PID_FILE="/tmp/hotbak_inc_mysql_${PORT}"
    LOCK_FILE="/var/lock/subsys/xtrabackup_${PORT}"
    
    function out_log ()
    {
            # process: out_log function
            # Syntax:  out_log
            # Author: Felix.li
            # Returns:
            #       N/A
    
            if [ -n "$1" ]; then
                    _PATH="$1"
            else
                    echo "unknown error"
                    echo -e "wrong\nunknown error" > ${MYSQL_LOG}
                    exit
            fi
    
            [ ! -f ${LOG_FILE} ] && touch "${LOG_FILE}"
            echo -e "[$(date +%Y-%m-%d' '%H:%M:%S)] ${_PATH}" >> "${LOG_FILE}"
    }
    
    function del_bak ()
    {
    
            for dbfile in `find "${1}/" -name "[0-9]*.tar.gz" -type f -mtime +${DAY}`; do
                    out_log "delete from ${dbfile}"
                    rm -f ${dbfile}
            done
    }
    
    # if ! rpm -q percona-xtrabackup > /dev/null 2>&1;then
    #     if [ `uname -r|grep -c el6.x86_64` -eq 1 ];then
    #             yum -y install perl-Time-HiRes perl-DBD-MySQL
    #             rpm -ivh http://122.226.74.168/percona-xtrabackup-2.1.3-608.rhel6.x86_64.rpm
    #     else
    #         rpm -q xtrabackup-1.6.4-313.rhel5 >/dev/null && rpm -e xtrabackup-1.6.4-313.rhel5
    #         rpm -q percona-xtrabackup-2.0.0-417.rhel5 >/dev/null && rpm -e percona-xtrabackup-2.0.0-417.rhel5
    #         rpm -q percona-xtrabackup-2.0.1-446.rhel5  >/dev/null || rpm -ivh --nodeps http://122.226.74.168/percona-xtrabackup-2.0.1-446.rhel5.x86_64.rpm >/dev/null
    #     fi
    # fi
    
    
    if [ ! -d "$BACKUP_PATH" ]; then
            out_log "mkdir -p ${BACKUP_PATH}"
            mkdir -p ${BACKUP_PATH}
            out_log "chown -R nobody.nobody ${BACKUP_PATH}"
            chown -R nobody.nobody ${BACKUP_PATH}
    fi
    
    [ ! -f $PID_FILE ] && touch ${PID_FILE}
    _PID=`cat ${PID_FILE}`
    if [ `ps ax|awk '{print $1}'|grep -v grep|grep -c "\b${_PID}\b"` -eq 1 ] && [ -f ${LOCK_FILE} ]; then
            echo -n $"xtrabackup process already exist."
            echo -e 'wrong\nxtrabackup process already exist.' >${MYSQL_LOG}
            exit
    else
            echo $$ >${PID_FILE}
            touch ${LOCK_FILE}
    fi
    
    function all_back() {
                    DB_NAME=("$HOST"_"$DATE"_"$PORT")
                    echo 'all'
                    out_log "cd ${BACKUP_PATH}"
                    cd ${BACKUP_PATH}
                    out_log "innobackupex ${BACKUP_PATH} --use-memory=${USE_MEM} --throttle=${IOPS} --host=${MYSQLHOST} --user=${USER} --port=${PORT} --password=${PASSWORD} --stream=tar --no-lock 2>>${LOG_FILE} | gzip - > ${BACKUP_PATH}/${DB_NAME}.tar.gz"
                    innobackupex ${BACKUP_PATH} --use-memory=${USE_MEM} --throttle=${IOPS} --host=${MYSQLHOST} --user=${USER} --port=${PORT} --password=${PASSWORD} --stream=tar --no-lock 2>>${LOG_FILE} | gzip - > ${BACKUP_PATH}/${DB_NAME}.tar.gz
    
                    out_log "tar zxfi ${BACKUP_PATH}/${DB_NAME}.tar.gz xtrabackup_checkpoints"
                    tar zxfi ${BACKUP_PATH}/${DB_NAME}.tar.gz xtrabackup_checkpoints
    
                    RETVAL=$?
    
                    if      [ ${RETVAL} -eq 0 -a `tail -50 "${LOG_FILE}" | grep -ic "\berror\b"` -eq 0 ]; then
                            del_bak "${BACKUP_PATH}"
                            echo -n $"Complete Hot Backup"
                            success
                            echo
                            echo -e "ok\n${DB_NAME}.tar.gz" > ${MYSQL_LOG}
                            echo -e "ok\n${DB_NAME}.tar.gz" > ${LAST_ALL_LOG}
                            rm -f ${LOCK_FILE}
                    else
                            out_log "[ERROR] error: xtrabackup failure"
                            echo -n $"Complete Hot Backup"
                            failure
                            echo
                            echo -e "wrong\n$(tail -50 "${LOG_FILE}" | grep -i "\berror\b" | sed -n '1p')" > ${MYSQL_LOG}
                            echo -e "wrong\n${DB_NAME}.tar.gz" > ${LAST_ALL_LOG}
                            rm -f ${LOCK_FILE}
                    fi
    }
    
    function inc_back {
                    CHECKPOINT=$(awk '/to_lsn/ {print $3}' ${BACKUP_PATH}/xtrabackup_checkpoints 2>/dev/null)
                    DB_NAME=("$HOST"_"$DATE"_"$PORT""-increase")
                    echo "inc"
    
                    if [ ! -e "${BACKUP_PATH}/xtrabackup_checkpoints" ];then
                    out_log "xtrabackup_checkpoints does not exist"
                    all_back
                    exit 0
            fi
    
            # if [ $(mysqladmin -u${USER} -p${PASSWORD} -h${MYSQLHOST} version |awk '/Server version/{split($3,array,".");print array[1]array[2]}') -eq 55 ]; then
                # XTRABACKUP="xtrabackup_55"
            # fi
            XTRABACKUP=xtrabackup
            out_log "${XTRABACKUP}  --backup --user=$USER --password=$PASSWORD --host=${MYSQLHOST}  --throttle=${IOPS} --target-dir=${BACKUP_PATH}/${DB_NAME} --incremental-lsn=${CHECKPOINT} --no-lock >>${LOG_FILE} 2>&1"
            ${XTRABACKUP} --backup --user=$USER --password=$PASSWORD --host=${MYSQLHOST} --throttle=${IOPS} --target-dir=${BACKUP_PATH}/${DB_NAME} --incremental-lsn=${CHECKPOINT} --no-lock >>${LOG_FILE} 2>&1
            RETVAL=$?
            out_log "cd ${BACKUP_PATH}"
            cd ${BACKUP_PATH}
            out_log "tar zcfi ${DB_NAME}.tar.gz ${DB_NAME} >/dev/null 2>>${LOG_FILE}"
            tar zcfi "${DB_NAME}.tar.gz" "${DB_NAME}" >/dev/null 2>>${LOG_FILE}
            out_log "rm -rf ${DB_NAME}"
            rm -rf ${DB_NAME}
    
            if  [ ${RETVAL} -eq 0 -a `tail -50 "${LOG_FILE}" | grep -ic "\berror\b"` -eq 0 ]; then
                del_bak "${BACKUP_PATH}"
                echo -n $"Incremental Hot Backup"
                success
                echo
                echo -e "ok\n${DB_NAME}.tar.gz" >${MYSQL_LOG}
                rm -f ${LOCK_FILE}
            else
                echo -n $"Incremental Hot Backup"
                failure
                echo
                echo -e "wrong\n$(tail -50 "${LOG_FILE}" | grep -i "\berror\b" | sed -n '1p')" >${MYSQL_LOG}
            rm -f ${LOCK_FILE}
            fi
    }
    
    if [ "${HOUR}" -eq "4" ];then
            all_back
    else
            inc_back
    fi
    

```

-   批量部署脚本样例
```
#!/usr/bin/env bash

# 写入配置文件
PUBIP=$(curl http://ip.mutantbox.online/getip.php|sed 's/\.//g')
GAMEID='12'
OPID='1'
REDIS_HOST='10.53.0.189'
REDIS_PORT='6381'
MYIP=$(ifconfig |grep -oP "10\.(\d+\.){2}\d+"|grep -v \.255$)
SID=`ls /data/code/game/ | head -n 1`
BAKNAME=`date "+%s"`


echo '
[global]
server_id = '''${SID}'''
game_id = '''${GAMEID}'''
op_id = '''${OPID}'''
interval = 3600
redis_host = 10.53.0.189
redis_port = 6381
redis_queue = uuzuback
my_ip = '''${MYIP}'''
log = /var/log/uuzu_back.log
error_log = /var/log/uuzu_back.error
' > /etc/uuzuback.conf


for i in `find  /data/apps/redis/conf/ -type f -name redis*.conf`;do
	redis_port=$(echo ${i} | grep -oP "\d+")
	echo '
[redis'''${redis_port}''']
back_type = redis
instance = '''${redis_port}'''
rsync_model = backup/redisbase_'''${redis_port}'''
back_dir = /data/backup/redisbase_'''${redis_port}'''/
back_log = /var/log/redis_'''${redis_port}'''.log
script = /bin/bash /usr/local/redis_back.sh /data/apps/redis/conf/redis'''${redis_port}'''.conf
' >> /etc/uuzuback.conf
done
	


#替换掉失效的
sed -i s/.*redis_save.*bash//g /var/spool/cron/root 

#安装相关组件
# yum install python-setuptools -y 
# yum install rsync xinetd -y
# yum remove python-redis -y 
# pip uninstall redis -y
# pip install redis
#查看python-redis版本
versionforpython=`python  -c "import redis;print redis.__version__"`
echo "python_redis_version ${versionforpython}"


#开启rsync模块
sed -i '/disable/ s#yes#no#g' /etc/xinetd.d/rsync

if [[ -f /etc/rsyncd.conf ]]; then
	/bin/cp -Rrf /etc/rsyncd.conf /etc/rsyncdbak_${BAKNAME}.conf
fi


cat>/etc/rsyncd.conf<<EOF
uid     = nobody
gid     = nobody
use chroot      = yes
max connections = 1000
pid file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log
list = no

[backup]
path = /data/backup/
hosts allow = 0.0.0.0/0
read only = yes

ignore errors = yes
EOF

/etc/init.d/xinetd restart


wget http://52.197.150.65:8889/scripts/redis_save/redis_backup.sh -O /usr/local/redis_back.sh
wget http://52.197.150.65:8889/scripts/redis_save/uuzuback_client.py -O /usr/local/uuzuback_client.py

#设置crontab
baktime=`echo $(($RANDOM%60))`
sed -i s/.*uuzuback_client.py//g /var/spool/cron/root
echo "$baktime */1 * * * /usr/bin/python  /usr/local/uuzuback_client.py"  >> /var/spool/cron/root 
sed -i '/^$/'d /var/spool/cron/root
service crond restart



```
