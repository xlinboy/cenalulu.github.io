```
1.前期准备
	1.hadoop集群
	2.mysql
2.mysql的安装
	1.只需要安装在集群里面的一台节点上即可，此处选择hadoop1节点
	2.在Hadoop1上安装mariadb
		yum -y install  mariadb-server   mariadb
	3.开启服务并开机自启
		systemctl start mariadb.service
		systemctl enable mariadb.service
	4.设置密码。第一次登陆时直接空密码登陆，之后使用sql语句设置密码
		mysql -u root -p
		登录之后，先查看databases是否正常，之后sql语句设置密码
		> use mysql;
		> update user set password=password( '123456' ) where user= 'root' ;
		然后设置root用户可以从任何主机登陆，对任何的库和表都有访问权限
		> grant all privileges on *.* to root@'%' identified by '123456';
		> grant all privileges on *.* to root@'hadoop1' identified by '123456';
		> grant all privileges on *.* to root@'localhost' identified by '123456';
		> FLUSH PRIVILEGES;
	5.修改mariadb的数据地址，真实集群节点中必要设置
		1.停止服务
			systemctl stop mariadb.service
		2.复制原来的配置到系统盘外的磁盘(举例是/data01)
			cp -r /var/lib/mysql/ /data01/
		3.备份原来的设置
			mv /var/lib/mysql/ /var/lib/mysql.bak/
		4.修改磁盘中文件夹的所属权限
			chown -R mysql:mysql /data01/mysql
		5.创建软连接
			ln -s /data01/mysql/ /var/lib/mysql
		6.重启Mariadb
			systemctl restart mariadb 

	7.所有节点安装mysql-connector驱动
		yum -y install mysql-connector-java
		安装之后的路径为/usr/share/java/mysql-connector-java.jar
	8.安装其他的依赖包
		yum -y install psmisc
		yum -y install perl
		yum -y install  nfs-utils  portmap
		systemctl start rpcbind
		systemctl enable rpcbind
	9.创建数据库和用户。
		后续需要用到数据库的组件，也在这里一并创建。这里创建hive、amon、hue、monitor和oozie。
		登陆数据库，执行以下语句，
		注意包含主机名的修改为自己的主机名
create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database amon DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database monitor DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database oozie DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive'; 
GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'localhost'; 
CREATE USER 'hive'@'%' IDENTIFIED BY 'hive'; 
GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'%'; 
CREATE USER 'hive'@'hadoop1'IDENTIFIED BY 'hive';  
GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'hadoop1';

CREATE USER 'oozie'@'localhost' IDENTIFIED BY 'oozie'; 
GRANT ALL PRIVILEGES ON oozie.* TO 'oozie'@'localhost';  
CREATE USER 'oozie'@'%' IDENTIFIED BY 'oozie'; 
GRANT ALL PRIVILEGES ON oozie.* TO 'oozie'@'%'; 
CREATE USER 'oozie'@'hadoop1'IDENTIFIED BY 'oozie';  
GRANT ALL PRIVILEGES ON oozie.* TO 'oozie'@'hadoop1'; 

CREATE USER 'monitor'@'localhost' IDENTIFIED BY 'monitor'; 
GRANT ALL PRIVILEGES ON monitor.* TO 'monitor'@'localhost';  
CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor'; 
GRANT ALL PRIVILEGES ON monitor.* TO 'monitor'@'%'; 
CREATE USER 'monitor'@'hadoop1'IDENTIFIED BY 'monitor';  
GRANT ALL PRIVILEGES ON monitor.* TO 'monitor'@'hadoop1'; 
FLUSH PRIVILEGES;  

3.Hive的安装，只需要在一个节点上安装(服务端)即可。
	1.下载软件包
		archive.apache.org/dist/
	2.上传到节点并解压
	3.配置环境变量，hive加入/etc/profile
	4.修改配置文件
		1.hive-env.sh中添加信息:
			export JAVA_HOME=...
			export HADOOP_HOME=...
			export HIVE_HOME=...
		2.hive-log4j.properties
			修改
			log4j.appender.EventCounter=org.apache.hadoop.log.metrics.EventCounter
		3.hive-site.xml：
			<configuration>
				<property>
					<name>javax.jdo.option.ConnectionURL</name>
					<value>jdbc:mysql://hadoop1:3306/hive?createDatabaseIfNotExist=true</value>
				</property>
				<property>
					<name>javax.jdo.option.ConnectionDriverName</name>
					<value>com.mysql.jdbc.Driver</value>
				</property>
				<property>
					<name>javax.jdo.option.ConnectionUserName</name>
					<value>root</value>
				</property>
				<property>
					<name>javax.jdo.option.ConnectionPassword</name>
					<value>123456</value>
				</property>
			</configuration>
	5.把hive/lib下的jline2.12拷贝到hadoop下的share/hadoop/yarn/lib/下，若存在旧版本就替换掉
	6.把mysql-connector这个jar包，拷贝到hive下的lib下

4.启动Hive
	hive

5.建表的时候显示键太长，是因为把字符编码都设置成了utf-8的原因，
	mysql > alter database hive character set latin1;即可解决
```	
