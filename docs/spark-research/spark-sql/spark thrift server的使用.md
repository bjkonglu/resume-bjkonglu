## Spark Thrift Server的使用

在我们使用SPARK SQL时，除了使用spark-sql.sh脚本实现对hive的查询外，我们还可以另一种选择，那就是Spark Thrift Server。这个方式方便我们对交互查询做二次开发，只要通过JDBC/ODBC的方式，便可以与Spark SQL应用进行交互，达到交互查询的功能。下面我们就开始介绍一下Spark Thrift Server的使用和原理。

### Spark Thrift Server的配置与启动

  首先，我们将hive的配置conf/hive-site.xml移动到spark的配置目录$SPARK_HOME/conf下，并增加下面几项配置，如下：
```scala
<configuration>
<!--spark-sql.sh使用的hive元数据库地址、用户名、密码和驱动配置-->
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://mysqlServerHost:port/hive_meta?useUnicode=true&amp;characterEncoding=UTF-8&amp;serverTimezone=Asia/Shanghai&amp;useSSL=false</value>
</property>
<property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
</property>
<property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hivemeta</value>
</property>
<property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>hivemeta</value>
</property>

<!--hive元数据存放地址, 被Spark Thrift Server用来获取hive元数据，使用之前必须确保hive启动了hive metastore服务（hive --service metastore）-->
<property>
    <name>hive.metastore.uris</name>
    <value>thrift://metaStoreHost:port</value>
    <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
</property>
 
<!--thrift server配置-->  
<property>
    <name>hive.server2.thrift.min.worker.threads</name>
    <value>5</value>
    <description>Minimum number of Thrift worker threads</description>
</property>
 
<property>
    <name>hive.server2.thrift.max.worker.threads</name>
    <value>500</value>
    <description>Maximum number of Thrift worker threads</description>
</property>
 
<property>
    <name>hive.server2.thrift.port</name>
    <value>thriftServerPort</value>
    <description>Port number of HiveServer2 Thrift interface. Can be overridden by setting $HIVE_SERVER2_THRIFT_PORT</description>
</property>
 
<property>
    <name>hive.server2.thrift.bind.host</name>
    <value>thriftServerHost</value>
    <description>Bind host on which to run the HiveServer2 Thrift interface.Can be overridden by setting$HIVE_SERVER2_THRIFT_BIND_HOST</description>
</property>
</configuration>
```
  然后，我们进行$SPARK_HOME/sbin，执行start-thriftserver.sh命令，如下。当命令执行完后，会在yarn集群上启动一个Spark SQL应用，这个应用的Driver端开启了一个thriftserver服务（相当一个RPC接口服务），外部可以通过访问这个接口服务，给上述Spark SQL应用传输执行命令，例如sql语句，这些命令由Spark引擎执行，并放回执行结果。
```shell
./start-thriftserver.sh --master yarn --excutor-memory 3g
```

  最后，我们需要一个客户端去访问启动的thriftserver服务，Spark自带一个客户端（beeline）。我们进入$SPARK_HOME/bin目录，执行beeline.sh脚本。
启动beeline客户端后，我们需要发出连接请求（!connect jbdc:hive2://thriftserverHost:port），去获取thriftserver的长连接（底层socket连接）。
当在客户端上执行指令（!sql sqlStatement）时，实际上的行为就是远程调用thrift接口。

### 扩展
  Spark Thrift Server实现了Spark SQL的连接，同时也是为我们可以开发符合自身业务特点的*即席查询*提供了思路。
