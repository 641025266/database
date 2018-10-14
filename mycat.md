## mycat的安装 

【应用场景】
```
主要是用来实现mysql的分库分表和读写分离
```

【服务器端安装过程】   
```shell
先要安装好jdk，最好是从官网下载源码后安装
tar -zxvf jdk-8u181-linux-x64.tar.gz -C /usr/local
vi  /etc/profile.d/java.sh 
export JAVA_HOME=/usr/local/jdk1.8.0_181
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH
source /etc/profile.d/java.sh
java -version
1.只实现分库分表
https://github.com/MyCATApache/Mycat-download 从该地址下载mycat的各个版本
wget http://dl.mycat.io/1.6-RELEASE/Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
192.168.112.130安装mycat，192.168.112.131-132作为mysql后端节点，用做分库分表，/etc/my.cnf里面binlog_format最好设置为row
tar -zxvf Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz -C /usr/local
cd /usr/local/mycat
vi conf/schema.xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
        <schema name="testdb" checkSQLschema="false" sqlMaxLimit="100">
           <table name="company" primaryKey="ID" autoIncrement="true" dataNode="dn1,dn2"  rule="mod-long" />
        </schema>
        <!-- 需要先在对应的mysql后端节点把db1和db2数据创建好，且把账号权限分配好，
             否则mycat启动不了 -->
        <dataNode name="dn1" dataHost="host1" database="db1" />
        <dataNode name="dn2" dataHost="host2" database="db2" />
        <dataHost name="host1" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <writeHost host="192.168.112.131" url="192.168.112.131:3306" user="root"
                                   password="123456">
                </writeHost>
        </dataHost>
        <dataHost name="host2" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <writeHost host="192.168.112.132" url="192.168.112.132:3306" user="root"
                                   password="123456">
                </writeHost>
        </dataHost>
</mycat:schema>
vi conf/rule.xml 
<mycat:rule xmlns:mycat="http://io.mycat/">
     中间有很多内容省略
     <function name="mod-long" class="io.mycat.route.function.PartitionByMod">
                     <!-- how many data nodes，因为后端只有两个数据库，所以count为2 -->
                     <property name="count">2</property>
     </function>
</mycat:rule>
vi  conf/server.xml
<mycat:server xmlns:mycat="http://io.mycat/">
        <user name="root">
                <property name="password">123456</property>
                <property name="schemas">testdb</property>
        </user>
原始配置文件这里还有一个readonly的user，如果没有这个用户就必须把该配置删除掉否则服务启动不了
</mycat:server>
 ./bin/mycat start
 ./bin/mycat status  确保服务是运行的
 tail -f logs/wrapper.log 确保日志没有任何错误出现，且出现success字样
 netstat -tunlp 查看8066服务端口和9066管理端口是否启动
 mysql -h 192.168.112.130 -u root -p123456 -P 8066 
Server version: 5.6.29-mycat-1.6-RELEASE-20161028204710 MyCat Server (OpenCloundDB)
注意此处已经是mycat的提示了
mysql> use testdb;
mysql> create table company(ID int not null auto_increment,name varchar(10),primary key(ID));
查看192.168.112.131-132的db1，db2库里面是否出现了company表
mysql -h 192.168.112.131 -u root -p123456
mysql> show tables from db1;
+---------------+
| Tables_in_db1 |
+---------------+
| company       |
mysql -h 192.168.112.132 -u root -p123456
mysql> show tables from db2;
+---------------+
| Tables_in_db2 |
+---------------+
| company       |
+---------------+
mysql -h 192.168.112.130 -u root -p123456 -P 8066 
mysql> use testdb;
mysql> INSERT INTO company(ID, NAME) values(1,'jerry');
mysql> INSERT INTO company(ID, NAME) values(2,'marry');
mysql> INSERT INTO company(ID, NAME) values(3,'mark');
查看数据是否被分开了
mysql -h 192.168.112.131 -u root -p123456
mysql> select * from db1.company;
+----+-------+
| ID | name  |
+----+-------+
|  2 | marry |
+----+-------+
mysql -h 192.168.112.132 -u root -p123456
mysql> select * from db2.company;
+----+-------+
| ID | name  |
+----+-------+
|  1 | jerry |
|  3 | mark  |
+----+-------+
2.只实现读写分离
192.168.112.130安装mycat，192.168.112.131当mysql的master，192.168.112.133当mysql的slave
准备工作：
192.168.112.131和133先配置好主从复制
cd /usr/local/mycat
vi conf/schema.xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
<schema name="test1" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn_test1"> </schema>
        <dataNode name="dn_test1" dataHost="dh_43" database="test1" />
        <dataHost name="dh_43" maxCon="1000" minCon="10" balance="1"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="131_M" url="192.168.112.131:3306" user="root"
                                   password="123456">
                        <!-- can have multi read hosts -->
                        <readHost host="133_S" url="192.168.112.133:3306" user="root" password="123456" />
                </writeHost>
        </dataHost>
</mycat:schema>
vi  conf/server.xml
<mycat:server xmlns:mycat="http://io.mycat/">
        <user name="root">
                <property name="password">123456</property>
                <!--  数据库名称要和schema.xml里面的一样 -->
                <property name="schemas">test1</property>
        </user>
vi   conf/log4j2.xml  为了查看读写分离效果，把log4j2.xml的level由info改为debug
<asyncRoot level="debug" includeLocation="true">
./bin/mycat start
./bin/mycat status  确保服务是运行的
tail -f logs/wrapper.log 确保日志没有任何错误出现，且出现success字样
netstat -tunlp 查看8066服务端口和9066管理端口是否启动
mysql -h 192.168.112.130 -u root -p123456 -P9066  登陆管理接口
mysql> show @@datasource\G;  查看读写分离配置情况
*************************** 1. row ***************************
  DATANODE: dn_test1
      NAME: 131_M
      TYPE: mysql
      HOST: 192.168.112.131
      PORT: 3306
       W/R: W
    ACTIVE: 0
      IDLE: 10
      SIZE: 1000
   EXECUTE: 411
 READ_LOAD: 0
WRITE_LOAD: 4
*************************** 2. row ***************************
  DATANODE: dn_test1
      NAME: 133_S
      TYPE: mysql
      HOST: 192.168.112.133
      PORT: 3306
       W/R: R
    ACTIVE: 0
      IDLE: 8
      SIZE: 1000
   EXECUTE: 410
 READ_LOAD: 6
WRITE_LOAD: 0
mysql> show @@heartbeat\G;   RS_CODE为1表示心跳正常
*************************** 1. row ***************************
            NAME: 131_M
            TYPE: mysql
            HOST: 192.168.112.131
            PORT: 3306
         RS_CODE: 1 
           RETRY: 0
          STATUS: idle
         TIMEOUT: 0
    EXECUTE_TIME: 2,3,20
LAST_ACTIVE_TIME: 2018-10-12 19:46:42
            STOP: false
*************************** 2. row ***************************
            NAME: 133_S
            TYPE: mysql
            HOST: 192.168.112.133
            PORT: 3306
         RS_CODE: 1
           RETRY: 0
          STATUS: idle
         TIMEOUT: 0
    EXECUTE_TIME: 2,3,26
LAST_ACTIVE_TIME: 2018-10-12 19:46:42
            STOP: false
mysql> use test1;            
mysql> create table table1(ID int not null auto_increment,name varchar(10),primary key(ID));
mysql> INSERT INTO table1(ID, NAME) values(1,'chen');
mysql> INSERT INTO table1(ID, NAME) values(2,'hao');
验证192.168.112.131和133是否都已经有数据产生了
mysql> select * from table1;
同时在另外一个终端监控wrapper.log
tail -f logs/wrapper.log |grep 'select read'
INFO   | jvm 1    | 2018/10/12 19:51:00 | 2018-10-12 19:51:00,518 [DEBUG][$_NIOREACTOR-0-RW] select read source 133_S for dataHost:dh_43  .....  表示读确实是去133_S的slave节点查询的
3.分库分表+读写分离+主从复制 
  就是把前面的分库分表，读写分离+主从复制的配置简单组合下就可以了

```

