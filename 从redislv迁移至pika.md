#从redislv迁移至pika
###安装pika
---
[下载地址](https://github.com/Qihoo360/pika/releases)
---
```
tar -jxvf pika-linux-x86_64-v3.1.0.tar.bz2
mv output/ /usr/local/pika
/usr/local/pika/bin/pika -c /usr/local/pika/conf/6381.conf
```
###安装迁移工具redis-port
---
[下载地址](https://main.qcloudimg.com/raw/47154504189a8941250f57b60f1e2fcb/redis-port.tgz)
---
```
mkdir /usr/local/redis-port
tar xzvf redis-port.tgz -C /usr/local/redis-port
```
###使用 redis-sync 在线迁移
```
cd /usr/local/redis-port/bin
./redis-sync -m 127.0.0.1:6382 -t 127.0.0.1:6381
[100.00%] #检测到此字段即为同步完成，完成之后进程不会退出，通过 Ctrl+C 命令或者其他方式终止工具的执行，即可停止数据同步。
```

###脚本使用
```
python redislv_to_pika.py -p xgame -P all -s 993 -c sync
python redislv_to_pika.py -p xgame -P all -s 993 -c set
```