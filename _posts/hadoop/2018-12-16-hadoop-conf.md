---
layout: article
title:  "hadoop安装与配制"
categories: hadoop
date:   2018-12-16 08:44:13
---

## 下载

&emsp;下载地址 `archive.apache.org/dist/hadoop/common/`  
&emsp;相关文档`http://hadoop.apache.org/docs`


### Linux相关配置  

##### 1. yum源
```
1. 上传包含安装包的文件到节点
2. 安装httpd包
rpm -ivh  apr-1.4.8-3.el7.x86_64.rpm 
rpm -ivh  apr-util-1.5.2-6.el7.x86_64.rpm 
rpm -ivh  httpd-tools-2.4.6-40.el7.centos.x86_64.rpm 
rpm -ivh  mailcap-2.1.41-2.el7.noarch.rpm
rpm -ivh  httpd-2.4.6-40.el7.centos.x86_64.rpm
3. 启动httpd进程，并设置为开机启动
systemctl   start    httpd
systemctl   enable   httpd
4. 安装createrepo包，用来构建软件仓库
rpm -ivh  deltarpm-3.6-3.el7.x86_64.rpm 
rpm -ivh  python-deltarpm-3.6-3.el7.x86_64.rpm 
rpm -ivh  libxml2-python-2.9.1-5.el7_1.2.x86_64.rpm 
rpm -ivh  libxml2-2.9.1-5.el7_1.2.x86_64.rpm 
rpm -ivh  createrepo-0.9.9-23.el7.noarch.rpm 
5.创建软链接到硬盘中的包文件夹
http工作目录是/var/www/html/，需要设置一个目录指向包文件夹的地址
cd  /var/www/html/
mkdir  /var/www/html/CentOS7.2/
ln -s  /data01/Packages   /var/www/html/CentOS7.2/Packages
chmod 755 -R /var/www/html/CentOS7.2/Packages
6.使用上面安装的createrepo来创建仓库
createrepo  /var/www/html/CentOS7.2/Packages
7.关闭防火墙
setenforce 0 
systemctl   stop    firewalld
访问IP地址下的/CentOS7.2/Packages能看到包并可以下载就可以了。
8.所有主机都配置一下repo文件
文件夹地址在/etc/yum.repos.d/
1.备份原来的repo源文件
cd /etc/yum.repos.d/
mkdir  bak
mv CentOS-*.repo  bak/
2.创建自己的repo文件，写入内容
vi  base.repo

[base]
name=CentOS-Packages
baseurl=http://192.168.33.201/CentOS7.2/Packages/
gpgkey=
path=/
enabled=1
gpgcheck=0
3.清理yum缓存，重新构建缓存，测试安装一个包
yum clean all
yum makecache
yum list|grep vim（举个例子，把包里面有vim的包都列出来）
```

##### 2. 禁用SELinux，否则重启之后可能会无法访问到yum的软件库

```
1.setenforce 0	临时关闭SELinux
2.修改/etc/selinux/config中的SELINUX=disabled
	sed -i 's/^SELINUX=.*/SELINUX=disabled/g'   /etc/selinux/config
	sed -i 's/^SELINUX=.*/SELINUX=disabled/g'   /etc/sysconfig/selinux
```

##### 3.关闭THP（影响hadoop集群的性能）
```
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

##### 4.修改swappiness
```
这个值表示剩余多少内存的时候使用磁盘，设置为0即最大限度的使用内存
echo "vm.swappiness=0" >> /etc/sysctl.conf 
sysctl -p    ##让配置生效
cat /proc/sys/vm/swappiness
```

##### 5.修改主机名，配置主机名文件，ssh免密登陆
```
1.修改主机名
	hostnamectl --static set-hostname hadoop1
2.配置主机名文件
	vi /etc/hosts
3.关闭防火墙即开机关闭
	systemctl stop firewalld.service
	systemctl disable firewalld.service
4.配置ssh免密登陆
	ssh-keygen
	ssh-copy-id 主机名

远程复制:
1. 复制文件（本地>>远程：scp /cloud/data/test.txt root@10.21.156.6:/cloud/data/
2. 复制文件（远程>>远程：scp root@10.21.156.6:/cloud/data/test.txt /cloud/data/
3. 复制目录（本地>>远程：scp -r /cloud/data root@10.21.156.6:/cloud/data/
4. 复制目录（远程>>本地：scp -r root@10.21.156.6:/cloud/data/ /cloud/data
```

##### 6.设置时区，统一时间
```
1.所有节点设置时区：
timedatectl set-timezone "Asia/Shanghai"
2.统一时间，“对表”，即以主节点的时间为准
1.所有机器安装ntp
	yum -y install ntp
2.修改主节点配置文件，所有机器时间以主节点（如hadoop1）时间为准，
	1.所有节点备份原始配置文件
		cp /etc/ntp.conf /etc/ntp.conf.bak
	2.修改主节点hadoop1配置
		vi /etc/ntp.conf
			# server 0.centos.pool.ntp.org iburst
			# server 1.centos.pool.ntp.org iburst
			# server 2.centos.pool.ntp.org iburst
			# server 3.centos.pool.ntp.org iburst 
			server 127.127.1.1
	3.重启ntpd进程，设置开机自启
		systemctl restart ntpd
		systemctl enable ntpd
3.在其他节点上指定以hadoop1为准来进行时间校准
	ntpdate hadoop1
	systemctl start ntpd
	crontab	定时执行脚本/命令
4.修改其他节点上的配置文件（同主节点修改步骤）
	vi /etc/ntp.conf
		# server 0.centos.pool.ntp.org iburst
		# server 1.centos.pool.ntp.org iburst
		# server 2.centos.pool.ntp.org iburst
		# server 3.centos.pool.ntp.org iburst 
		server 192.168.33.201
5.其他节点上重启ntpd进程，并设置成开机自启
	systemctl restart ntpd
	systemctl enable ntpd
```

## Hadoop相关
1. 安装jdk
```
JAVA_HOME
CDH中spark会默认到/usr/java/default目录下去找jdk，所以一般就安装在/usr/java目录下
export JAVA_HOME=/usr/java/jdk1.8.0_25
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$JAVA_HOME/bin:$PATH
```
2. 解压到指定目录
3. 修改配置文件
```
1.配置jdk的地址，etc/hadoop/hadoop-env.sh文件
	# set to the root of your Java installation
	export JAVA_HOME=/usr/java/latest
	
standalone本地运行的操作
	  $ mkdir input
	  $ cp etc/hadoop/*.xml input
	  $ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.6.jar wordcount input output
	  $ cat output/*
	  
2.配置hdfs，etc/hadoop/core-site.xml文件
新建文件夹data/tmp，用来存放临时文件
<configuration>
	<property>
		<name>fs.defaultFS</name>
		## 指定了namenode的节点
		<value>hdfs://hadoop1:9001</value>
	</property>
	<property>
		<name>hadoop.tmp.dir</name>
		<value>/opt/modules/hadoop-2.7.7/data/tmp</value>
	</property>
</configuration>
		
	etc/hadoop/hdfs-site.xml文件，配置副本数
	<configuration>
		<property>
			<name>dfs.replication</name>
			<value>1</value>
		</property>
		# python操作hdfs时需要获取权限
		<property> 
			<name>dfs.permissions</name> 
			<value>false</value> 
		</property>
		## 指定secondarynamenode节点
		<property>
			<name>dfs.namenode.secondary.http-address</name>
			<value>节点地址:50090</value>
		</property>
	</configuration>
	
	3.首次启动，namenode需要格式化
		bin/hdfs namenode -format
		
	hdfs的web监控端口是50070
			
    4.配置yarn
    etc/hadoop/mapred-site.xml文件
    	<configuration>
    		<property>
    			<name>mapreduce.framework.name</name>
    			<value>yarn</value>
    		</property>
    		## 配置historyserver
    		<property>
    			<name>mapreduce.jobhistory.address</name>
    			<value>节点主机名:10020</value>
    		</property>
    		<property>
    			<name>mapreduce.jobhistory.webapp.address</name>
    			<value>节点主机名:19888</value>
    		</property>
    	</configuration>
    
    etc/hadoop/yarn-site.xml文件
    	<configuration> 
        	<property>
        		<name>yarn.nodemanager.aux-services</name>
        		<value>mapreduce_shuffle</value>
        	</property>
        	## 指定resourcemanager节点
        	<property>
        		<name>yarn.resourcemanager.hostname</name>
        		<value>主节点的hostname</value>
        	</property>
    		## 日志聚集功能开启
    		<property>
    			<name>yarn.log-aggregation-enable</name>
    			<value>true</value>
    		</property>
    		## 日志文件保存的时间，以秒为单位
    		<property>
    			<name>yarn.log-aggregation.retain-seconds</name>
    			<value>640800</value>
    		</property>
    	</configuration>
			
    slaves: 指定启动的脚本，如启动start-dfs.sh
    	指定datanode和nodemanager是哪些节点
    yarn的web监控端口是8088
			
	5.配置文件的类别
		1.默认配置文件:四个模块相对应的jar中，$HADOOP_HOME/share/hadoop/...
			core-default.xml
			hdfs-default.xml
			yarn-default.xml
			mapred-default.xml
		2.自定义配置文件，$HADOOP_HOME/etc/hadoop/
			core-site.xml
			hdfs-site.xml
			yarn-site.xml
			mapred-site.xml
			
	6.启动方式
		1.各个节点服务组件逐一启动
			hadoop-daemon.sh start namenode
			hadoop-daemon.sh start datanode
			sbin/mr-jobhistory-daemon.sh start historyserver
		2.各个模块统一启动
			hadoop1:start-dfs.sh
			hadoop2:start-yarn.sh	mr-jobhistory-daemon.sh start historyserver
				
		3.所有节点服务统一启动
			start-all.sh（不建议使用）
			因为一般实际情况下namenode和resourcemanager不建议配置在一台机器上
	
	7.查看安全模式
		hdfs dfsadmin -safemode get
		
	8.其他
		yarn默认的配置，cpu有8核，内存为8G，可以配置
			
		apache slider 动态的yarn
			把已经存在的分布式框架运行在yarn上
		
	9.mr
		以wordcount为例：
		map：映射，可以高度并行
			输入输出形式都是键值对
				输入的key，value对，key为字符偏移量，value为字符串类型
				
		reduce：合并
		
		mapreduce优化
			1.reduce的个数，job.reduces 默认为1个
			2.shuffle过程的compress和combiner
			3.shuffle的各种参数调节
```

上传到工作目录:

```
hdfs dfs -put /etc/profile /file
```


​		
​		
​		
​		
​		
​		
​		
​		
​		
​	