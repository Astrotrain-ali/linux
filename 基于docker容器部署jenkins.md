#基于docker容器部署jenkins
#####安装
	yum install -y docker-ce
#####下载镜像
	docker pull jenkinsci/jenkins:2.150.1-alpine
#####创建宿主机目录
	mkdir /data/jenkins-workspace
#####权限
	chmod 777 /data/jenkins-workspace
#####运行
	docker run -p 8090:8080 -v /data/jenkins-workspace/:/var/jenkins_home jenkinsci/jenkins:2.150.1-alpine
	终端会打印安装密钥和密钥文件This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
	我这里的密钥是48b45c893f33485993595c905c85feff
#####后台运行
	docker run -d -p 8090:8080 -v /data/jenkins-workspace/:/var/jenkins_home -v /etc/localtime:/etc/localtime -v /etc/timezone:/etc/timezone jenkinsci/jenkins:2.150.1-alpine
	-d 后台运行
	8090宿主机监听端口，8080容器监听端口，此处用意是把容器的8080映射到宿主机的8090端口对外提供服务
	/data/jenkins-workspace/宿主机工作目录，/var/jenkins_home容器工作目录，此处用意同端口映射道理
	jenkinsci/jenkins:2.150.1-alpine 镜像名称
#####页面安装
	1，浏览器打开ip:8090
	2，填入密钥即可48b45c893f33485993595c905c85feff
	3，选择插件
	

#####配置jenkins容器可以免密登录宿主机
ps:想让容器免密登录哪台机器 就把公钥配置到哪一台主机即可

-	进入容器运行shell,生成密钥，然后将公钥追加至宿主机

		docker exec -it fervent_cerf /bin/bash
		mkdir ~/.ssh && cd ~/.ssh
		ssh-keygen -t rsa #一直回车即可
		cat id_rsa.pub  #复制这个公钥追加到宿主机的~/.ssh/authorized_keys文件中



####修复jenkins容器时间和宿主机不同步问题
	删除容器，重新启动的时候挂载宿主机的时区文件
	docker rm -f b8bbfdf36c8f
	docker run -d --name jenkins -p 8090:8080 -v /data/jenkins-workspace/:/var/jenkins_home -v /etc/localtime:/etc/localtime -v /etc/timezone:/etc/timezone jenkinsci/jenkins:2.150.1-alpine 





jenkins帐号
admin
KiRdc5LKynC4