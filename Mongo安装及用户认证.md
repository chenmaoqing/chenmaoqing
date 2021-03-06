# mongo安装和使用
 #
安装mongo

参考文档：https://docs.mongodb.com/manual/tutorial/install-mongodb-enterprise-on-red-hat/

### yum安装，先获取repo源 ###

	[mongodb-enterprise]
	name=MongoDB Enterprise Repository
	baseurl=https://repo.mongodb.com/yum/redhat/$releasever/mongodb-enterprise/3.6/$basearch/
	gpgcheck=1
	enabled=1
	gpgkey=https://www.mongodb.org/static/pgp/server-3.6.asc

sudo yum install -y mongodb-enterprise  #安装最新mongo，默认会依赖安装下面的包

	mongodb-enterprise,
	mongodb-enterprise-server,
	mongodb-enterprise-shell,
	mongodb-enterprise-mongos,
	mongodb-enterprise-tools
	
安装指定版本使用下面命令

	yum install -y mongodb-enterprise-3.6.5 mongodb-enterprise-server-3.6.5 mongodb-enterprise-shell-3.6.5 mongodb-enterprise-mongos-3.6.5 mongodb-enterprise-tools-3.6.5

启动使用

	systemctl start mongod
	service mongod start/stop/restart
	chkconfig mongod on
	mongo --host 127.0.0.1:27017

数据目录

	日志：/var/log/mongodb
	数据：/var/lig/mongo
	

### 使用resource安装 ###

获取Mongo的安装包： 

	curl -O https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-3.4.6.tgz
	tar -xf mongodb-linux-x86_64-3.4.6.tgz
	mv mongodb-3.4.6 /usr/local/mongodb
	cd /usr/local/mongodb
	手动创建db和log目录
	mkdir -p /data/db
	mkdir /data/log
	touch /data/log/mongodb.log

启动mongo

	./bin/mongod  --dbpath /data/db --logpath /data/log/mongodb.log --fork --port 27017
	--dbpath 数据存储目录
	--logpath mongo运行日志记录
	--fork 是后台运行
	--port  是运行的端口（默认是27017）
![mongo1](/img/mongo1.jpg)

出现如上图的字样就说明启动成功了


进入数据库

	./bin/mongo  

![mongo2](/img/mongo2.jpg)

### mongodb数据简介 ###
	mongodb是一个介于nosql数据库和mysql数据库之间的一个数据存储系统，它没有严格的数据格式，
	但同时支持复杂查询，而且自带sharding模式和Replica Set模式，支持分片模式，复制模式，
	自动故障处理，自动故障转移，自动扩容，全内容索引，动态查询等功能。扩展性和功能都比较强大。

    mongodb在数据查询方面，支持类sql查询，可以一个key多value内容，可以组合多个value内容来查询，
	支持索引，支持联合索引，支持复杂查询 ，支持排序，基本上除了join和事务类型的操作外，
	mongodb支持所有mysql支持的查询，甚至某个客户端api支持直接使用sql语句查询mongodb。

    mongodb的sharding功能目前日渐完善，支持自定义范围分片，hash自动分片等，分片自动扩容，
	shard之间自动负载均衡等功能。实际使用中功能还不错。

	mongodb 文档数据库,存储的是文档(Bson->json的二进制化).
	特点:内部执行引擎为JS解释器, 把文档存储成bson结构,在查询时,转换为JS对象,并可以通过熟悉的js语法来操作.

	mongo和传统型数据库相比,最大的不同:
	传统型数据库: 结构化数据, 定好了表结构后,每一行的内容,必是符合表结构的,就是说--列的个数,类型都一样.

	mongo文档型数据库: 表下的每篇文档,都可以有自己独特的结构(json对象都可以有自己独特的属性和值)

安装目录下，bin下脚本作用

![mongo3](/img/mongo3.png)


##mongo数据库的用户验证##
	超级账号创建
	> db.createUser(
		... ... {
		... ...    user:"admin",
		... ...    pwd:"admin123",
		... ...    roles:[ { role:"userAdminAnyDatabase",db:"admin"}]
		... ... } )


	创建完成后，修改mongo启动的配置文件，加入auth=on
	dbpath=/data/db
	logpath=/data/log/mongodb.conf
	port=27017
	auth=on
	fork=true
	logappend=true
	
	杀死mongo的所有进程，
	pkill -9 mongo  

	重新启动mongod(server)

	[root@www1 mongodb]# ~/bin/mongod -f conf/mongodb.conf 
	about to fork child process, waiting until server is ready for connections.
	forked process: 18135
	child process started successfully, parent exiting
	
	[root@www1 mongodb]# ./bin/mongo  启动mongo(client)

	> show dbs
	2018-06-24T21:47:19.974+0800 E QUERY    [thread1] Error: listDatabases failed:{
		"ok" : 0,
		"errmsg" : "not authorized on admin to execute command { listDatabases: 1.0, $db: \"admin\" }",
		"code" : 13,
		"codeName" : "Unauthorized"
	} :


	这时无法show dbs就没有权限查看数据库了，需要使用admin来验证登陆
	> db.auth('admin','admin123')
	1
	返回1表示已经验证成功 
	> show dbs
	admin   0.000GB
	config  0.000GB
	data    0.000GB
	local   0.000GB

	此时admin验证成功，但是每个数据库此时会要有自己的认证用户
	若要查看某一个数据库的数据，还要使用admin超管为每个数据库创建账号，并认证登陆
	需要注意的是在建立data数据库用户的时候一定要先启用data数据库，否则会出现问题
	>use data
	>db.createUser({user:'u1',pwd:'123qwe',roles:[{role:'readWrite',db:'data'}]})
	> db.auth('u1','123qwe')
	1
	> show collections
	stu

超管密码忘记，更改密码步骤：

1、更改配置文件，将auth=true注释掉，或者true改为false

2、重启mongo

	pkill -9 mongo  
	./bin/mongod -f conf/mongodb.conf
	./bin/mongo
	>use admin
	> db.system.users.find()  #查找admin用户
	> db.system.users.remove({'_id':'data.admin'}) #根据id将admin用户删除，然后重新建admin
	> 
	> db.createUser(
		... ... {
		... ...    user:"admin",
		... ...    pwd:"admin123",
		... ...    roles:[ { role:"userAdminAnyDatabase",db:"admin"}]
		... ... } )
	db.createUser({ user:"admin",pwd:"admin123", roles:[{role:"userAdminAnyDatabase",db:"admin"}]})


3、再次kill掉mongo，将auth改为true后进行重启
	
## mongo数据库Role ##
	
	Built-In Roles（内置角色）：
	1. 数据库用户角色：read、readWrite;
	2. 数据库管理角色：dbAdmin、dbOwner、userAdmin；
	3. 集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；
	4. 备份恢复角色：backup、restore；
	5. 所有数据库角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
	6. 超级用户角色：root  
	// 这里还有几个角色间接或直接提供了系统超级用户的访问（dbOwner 、userAdmin、userAdminAnyDatabase）
	7. 内部角色：__system

	Read：允许用户读取指定数据库
	readWrite：允许用户读写指定数据库
	dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile
	userAdmin：允许用户向system.users集合写入，可以找指定数据库里创建、删除和管理用户
	clusterAdmin：只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限。
	readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限
	readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限
	userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限
	dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限。
	root：只在admin数据库中可用。超级账号，超级权限

	userAdminAnyDatabase 权限只是针对用户管理的，对其他是没有权限的。
	mongodump --port=27020 -uzjyr -pzjyr --db=test -o backup   
	#只要读权限就可以备份
	mongorestore --port=27020 -uzjy -pzjy --db=test backup/test/  
	#读写权限可以进行还原


	更新用户密码
		use xx
		db.changeUserPassword("username","newpassword")

	删除用户
		切换到用户授权的db
		use xx
		执行删除操作
		db.dropUser("username")
	更新用户
		切换到用户授权的db
		use xx
		执行更新
		字段会覆盖原来的内容

		db.updateUser("username",{
		    pwd:"new password",
		    customData:{
		        "title":"PHP developer"
		    }
		})

	查看角色信息
		use admin
		db.getRole("rolename",{showPrivileges:true})
	
	删除角色
		use admin
		db.dropRole("rolename")更新用户密码

	查看用户信息
		use admin
		db.getUser("username")
