# CentOS6 升级GCC版本至4.9.3
GCC

出现报错
今天线上测试机使用node安装项目所需组件报错为

线上测试环境 
node : v8.4.0 
npm : v5.6.0 
gcc : yum源中默认的4.4.7版本

[root@localhost ~]# npm install ethereumjs-accounts
... ...
OSError: [Errno 13] Permission denied
gyp ERR! configure error 
gyp ERR! stack Error: `gyp` failed with exit code: 1
... ...
查找问题原因
EXM？使用root用户执行的命令，居然报错Permission denied??? 
然而，这组件就是死活装不上 
机智的拿着报错贴给谷歌看一哈，果然有很多人遇到这个问题。解决方案就是必须提升系统的gcc版本到4.8以上才行

解决问题
既然知道了问题所在，那就只能升级系统上的gcc版本咯。当前系统为CentOS 6.9，查看官网得知，可安装的gcc版本最高为4.9+

源码编译安装GCC 4.9.3
升级安装部分依赖包
[root@localhost ~]# yum install libmpc-devel mpfr-devel gmp-devel
下载对应的GCC源码包
[root@localhost ~]# cd /usr/src/
[root@localhost ~]# curl ftp://ftp.mirrorservice.org/sites/sourceware.org/pub/gcc/releases/gcc-4.9.3/gcc-4.9.3.tar.bz2 -O
[root@localhost ~]# tar xvfj gcc-4.9.3.tar.bz2
利用GCC自带的工具下载依赖包
[root@localhost ~]# cd gcc-4.9.3
[root@localhost ~]# ./contrib/download_prerequisites
该命令可以下载当前版本的gcc所依赖的环境包，并解压在当前目录中。 
如果当前主机无法联网，可以打开该文件，将该文件中的wget命令后的安装包手动下载上传后，解压至/usr/src/gcc-4.9.3目录下，再将该脚本中的wget行注释掉，重新执行一次脚本即可

编译安装GCC
[root@localhost ~]# ./configure --disable-multilib --enable-languages=c,c++
[root@localhost ~]# make -j `grep processor /proc/cpuinfo | wc -l`
[root@localhost ~]# make install
添加环境变量
[root@localhost ~]# vim ~/.bashrc 或者 vim ~/.zshrc
export LD_LIBRARY_PATH=/usr/local/lib:/usr/local/lib64/:$LD_LIBRARY_PATH
export C_INCLUDE_PATH=/usr/local/include/:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=/usr/local/include/:$CPLUS_INCLUDE_PATH
[root@localhost ~]# source ~/.bashrc 或者 source ~/.zshrc
验证GCC版本
[root@localhost ~]# gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/local/libexec/gcc/x86_64-unknown-linux-gnu/4.9.3/lto-wrapper
Target: x86_64-unknown-linux-gnu
Configured with: ./configure --disable-multilib --enable-languages=c,c++
Thread model: posix
gcc version 4.9.3 (GCC)
yum源安装GCC 4.9+
当前的GCC版本

# root @ elk1 in ~ [12:06:18]
$ gcc -v
使用内建 specs。
目标：x86_64-redhat-linux
配置为：../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-languages=c,c++,objc,obj-c++,java,fortran,ada --enable-java-awt=gtk --disable-dssi --with-java-home=/usr/lib/jvm/java-1.5.0-gcj-1.5.0.0/jre --enable-libgcj-multifile --enable-java-maintainer-mode --with-ecj-jar=/usr/share/java/eclipse-ecj.jar --disable-libjava-multilib --with-ppl --with-cloog --with-tune=generic --with-arch_32=i686 --build=x86_64-redhat-linux
线程模型：posix
gcc 版本 4.4.7 20120313 (Red Hat 4.4.7-18) (GCC)
开始升级gcc版本

# root @ elk1 in ~ [12:06:18]
$ yum install centos-release-scl -y  ## 安装所需要的yum源
# root @ elk1 in ~ [12:06:18]
$ yum install devtoolset-3-toolchain -y  ## 会安装gcc和gcc-c++ 4.9.2版本
# root @ elk1 in ~ [12:06:18]
$ scl enable devtoolset-3 bash/zsh  ## 为bash或者zsh启用gcc 4.9.2
验证当前gcc版本

# root @ elk1 in ~ [12:26:54] C:4
$ gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/opt/rh/devtoolset-3/root/usr/libexec/gcc/x86_64-redhat-linux/4.9.2/lto-wrapper
Target: x86_64-redhat-linux
Configured with: ../configure --prefix=/opt/rh/devtoolset-3/root/usr --mandir=/opt/rh/devtoolset-3/root/usr/share/man --infodir=/opt/rh/devtoolset-3/root/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --enable-multilib --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --enable-languages=c,c++,fortran,lto --enable-plugin --with-linker-hash-style=gnu --enable-initfini-array --disable-libgcj --with-isl=/builddir/build/BUILD/gcc-4.9.2-20150212/obj-x86_64-redhat-linux/isl-install --with-cloog=/builddir/build/BUILD/gcc-4.9.2-20150212/obj-x86_64-redhat-linux/cloog-install --with-mpc=/builddir/build/BUILD/gcc-4.9.2-20150212/obj-x86_64-redhat-linux/mpc-install --with-tune=generic --with-arch_32=i686 --build=x86_64-redhat-linux
Thread model: posix
gcc version 4.9.2 20150212 (Red Hat 4.9.2-6) (GCC)




