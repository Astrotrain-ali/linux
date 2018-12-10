##mysql ibdata1 瘦身
#####MySql innodb如果是共享表空间，ibdata1文件越来越大,mysql ibdata1存放数据，索引等，是MYSQL的最主要的数据。如果不把数据分开存放的话，这个文件的大小很容易就上了G，甚至几十G。对于某些应用来说，并不是太合适。因此要把此文件缩小。
#####无法自动收缩，必须数据导出，删除ibdata1，然后数据导入，比较麻烦，因此需要改为每个表单独的文件。
#####解决方法：数据文件单独存放(共享表空间如何改为每个表独立的表空间文件)。
##步骤如下：
###1)备份数据库
	mysqldump  --all-databases > all_database_201801210.sql
###2)停掉数据库
	systemctl stop mariadb
###3)修改my.cnf配置文件
	innodb_file_per_tab=1
###4）删除数据库目录下相关文件
	ibdata1
	ib_logfile*
	mysql-bin.index	
###5）手动删除除Mysql之外所有数据库文件夹，然后启动数据库
	systemctl start mariadb
###6）还原数据
 	mysql -uroot -p123456 < all_database_201801210.sql