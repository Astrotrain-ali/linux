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
三台机器做为集群，
centos6.5






