# HAProxy + MyCat + MySQL 集群部署

## 架构图

![use3.png](http://upload-images.jianshu.io/upload_images/732352-6864bdb92e1bc865.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 准备节点

### MySQL

1. mysql_master 
	
	```
	ip: 172.17.0.6
	server-id: 2101
	```
	
2. mysql_slave_01
	
	```
	ip: 172.17.0.7
	server-id: 2102
	```

3. mysql_slave_02

	```
	ip: 172.17.0.8
	server-id: 2103
	```
	
### MyCat

```
mycat_01 ip: 172.17.0.9
mycat_02 ip: 172.17.0.10
```
mycat 模拟用户

```
管理员：root:123456
只读：guest:123456
```
### HAProxy

```
ip: 172.17.0.11
```


## MySQL 主从复制

参考[Amoeba for MySQL](http://www.jianshu.com/p/4aec9f682509)

## HAProxy

参考 [RabbitMQ 高可用集群](http://www.jianshu.com/p/97fbf9c82872)


## 部署 MyCat

1. 下载 MyCat
	
	```
	root@mycat_02:~# wget http://dl.mycat.io/1.6-RELEASE/Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
	root@mycat_02:~# ls
	Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
	root@mycat_02:~# cd /usr/local/
	root@mycat_02:/usr/local# tar -zxvf /root/Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
	```

2. 配置 MyCat
	
	server.xml
	
	```
	<user name="root">
	<property name="password">123456</property>
	<property name="schemas">test</property>
	</user>
	
	<user name="guest">
		<property name="password">123456</property>
		<property name="schemas">test</property>
		<property name="readOnly">true</property>
	</user>
	```
	
	schema.xml
	
	```
	<?xml version="1.0"?>
	<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
	<mycat:schema xmlns:mycat="http://io.mycat/">
	
		<schema name="test" checkSQLschema="false" sqlMaxLimit="100" dataNode='dnTest'>
		</schema>
		<dataNode name="dnTest" dataHost="robostore" database="test" />
		<dataHost name="robostore" maxCon="1000" minCon="10" balance="1"
				  writeType="0" dbType="mysql" dbDriver="native" switchType="2"  slaveThreshold="100">
			<heartbeat>show slave status</heartbeat>
			<!-- can have multi write hosts -->
			<writeHost host="master" url="172.17.0.6:3306" user="root"
					   password="root">
				<!-- can have multi read hosts -->
				<readHost host="slave01" url="172.17.0.7:3306" user="root" password="root" />
				<readHost host="slave02" url="172.17.0.8:3306" user="root" password="root" />
			</writeHost>
		</dataHost>
	</mycat:schema>
	```
	
	**属性解释**
	
	balance:
	
	```
	0: 不开启读写分离机制。
	1: 全部的readHost与stand by writeHost参与select语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且M1与 M2互为主备)，正常情况下，M2,S1,S2都参与select语句的负载均衡。
	2. 所有的readHost与writeHost都参与select语句的负载均衡，也就是说，当系统的写操作压力不大的情况下，所有主机都可以承担负载均衡。
	```
	
	writeType：
	
	```
	0: 写操作只写入一个节点
	```
	
	switchType:
	
	```
	-1: 表示不启用主从切换
	1: 为默认值，自动切换
	2: 基于主从同步的状态，决定是否切换，与show slave status心跳对应
	3: 基于多住galary集群切换，与show status like 'wsrep%'对应
	```

3. 运行 MyCat

    ```
    root@mycat_02:/usr/local/mycat/bin# ./mycat start
    ```

