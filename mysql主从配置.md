##mysql主从配置

**主配置**

	[client]
	port            = 3306
	socket          = /tmp/mysql.sock
	
	[mysqld]
	user            = mysql
	datadir         = /data/mysql3306
	port            = 3306
	socket          = /tmp/mysql.sock
	basedir         = /usr/local/mysql
	pid-file        = /data/mysql3306/mysql.pid
	
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
	
	server_id = 85
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
	innodb_buffer_pool_size = 4G
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



	>GRANT REPLICATION SLAVE ON *.* TO 'slave'@'172.31.21.109' IDENTIFIED BY 'edJelrZuAgpNeLyn';

	>FLUSH PRIVILEGES







**从配置**

	[client]
	port            = 3306
	socket          = /tmp/mysql.sock
	
	[mysqld]
	user            = mysql
	datadir         = /data/mysql3306
	port            = 3306
	socket          = /tmp/mysql.sock
	basedir         = /usr/local/mysql
	pid-file        = /data/mysql3306/mysql.pid
	
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
	
	server_id = 156
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
	innodb_buffer_pool_size = 1G
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



**主节点执行：**

	show master status;\G
	记录pos位置

**从节点执行**

	change master to master_host='172.31.28.45',master_user='slave',master_password='edJelrZuAgpNeLyn',
	         master_log_file='mysql-bin.000002',master_log_pos=191; 
	
	start slave;