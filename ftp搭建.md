#基于系统用户的ftp
一、创建系统用户
    
    groupadd  xgameusr
    useradd  xgameusr  -g xgameusr  -s /sbin/nologin -d /data/ftp_server
    echo 'JBXR+mxxxR4jK9x3j' | passwd xgameusr --stdin
    
二、安装ftp服务

    yum install  vsftpd -y
    
三、设置ftp用户

    #解释一下,这个文件里的用户都不能登录
    cat /etc/passwd|awk -F : '{print $1}' |grep -v xgameusr > /etc/vsftpd/user_list
    
四、创建ftp目录并设置权限

    mkdir -p  /data/ftp_server/wwwroot
    chown -R xgameusr. /data/ftp_server    

五、挂载到本地,设开机启动

    mount --bind /data/wwwroot  /data/ftp_server/wwwroot/
    systemctl start Vsftpd
    systemctl enable Vsftpd
    echo 'mount --bind /data/wwwroot  /data/ftp_server/wwwroot/' >>  /etc/rc.local 

六、简单配置

    vim /etc/vsftpd/vsftpd.conf
    anonymous_enable=NO
    local_enable=YES
    write_enable=NO
    local_umask=022
    dirmessage_enable=YES
    xferlog_enable=YES
    connect_from_port_20=YES
    xferlog_std_format=YES
    listen=YES
    pam_service_name=vsftpd
    userlist_enable=YES
    tcp_wrappers=YES
    
七、挂载到远程

    1.装这个centos 可能需要一个软件源,根据自己系统版本到网上找
    rpm -Uhv rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm
    yum install curlftpfs
    2.创建ftp目录
    mkdir -p  /data/ftp_server/wwwroot

    3.列如:
    curlftpfs -o codepage=utf8 ftp://username:password@192.168.1.11 /ftp
    curlftpfs -o rw,allow_other ftp://xgameusr:JBXR+mxxxR4jK9x3j@182.254.208.180  /data/ftp_server/wwwroot


#基于虚拟用户的ftp配置，将用户限制在指定目录中
一、主配置文件

	vsftpd.conf
	anonymous_enable=NO
	local_enable=YES
	write_enable=YES
	local_umask=022
	dirmessage_enable=YES
	xferlog_enable=YES
	
	connect_from_port_20=NO
	pasv_enable=YES
	pasv_min_port=40500
	pasv_max_port=40600
	
	xferlog_file=/var/log/vsftpd.log
	
	xferlog_std_format=YES
	
	listen=YES
	
	guest_enable=YES
	guest_username=xgameusr
	pam_service_name=vsftpd
	userlist_enable=YES
	
	user_config_dir=/etc/vsftpd/vsftpd_user_conf
二、每一个虚拟用户的配置

	vsftpd_user_conf/xgameusr 
	local_root=/data/ftpdir/gqxd
	anon_world_readable_only=NO
	write_enable=YES
	anon_mkdir_write_enable=YES
	anon_upload_enable=YES
	anon_other_write_enable=YES
三、虚拟用户的用户名和密码配置

	vim vuser.list 
	sgbs    # 用户名
	sgbs1!@  # 密码

四、然后添加用户之后 将用户文本信息文件转换为db数据库并使用hash加密 执行
 	db_load -T -t hash -f /etc/vsftpd/vuser.list /etc/vsftpd/vsftpd_login.db