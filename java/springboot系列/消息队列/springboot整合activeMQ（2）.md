---
id="2018-09-06-10-38"
title="springboot整合ActiveMQ（2）"
headWord="接着上文来说，这里来说如何实现activemq的主从备份"
tags=["java", "spring","springboot","消息队列","activeMQ"]
category="java"
serie="spring boot学习"
---
[id]:2018-09-06
[type]:javaee
[tag]:java,spring,activemq


&emsp;&emsp;单个MQ节点总是不可靠的，一旦该节点出现故障，MQ服务就不可用了，势必会产生较大的损失。这里记录activeMQ如何开启主从备份，一旦master（主节点故障），slave（从节点）立即提供服务，实现原理是运行多个MQ使用同一个持久化数据源，这里以jdbc数据源为例。同一时间只有一个节点（节点A）能够抢到数据库的表锁，其他节点进入阻塞状态，一旦A发生错误崩溃，其他节点就会重新获取表锁，获取到锁的节点成为master，其他节点为slave，如果节点A重新启动，也将成为slave。

​	主从备份解决了单节点故障的问题，但是同一时间提供服务的只有一个master，显然是不能面对数据量的增长，所以需要一种横向拓展的集群方式来解决面临的问题。

### 一、activeMQ设置

#### 1、平台版本说明：

- 平台：windows
- activeMQ版本：5.9.1，[下载地址](https://www.apache.org/dist/activemq/5.9.1/apache-activemq-5.9.1-bin.zip.asc)
- jdk版本：1.8

#### 2、下载jdbc依赖

&emsp;&emsp;下载下面三个依赖包，放到activeMQ安装目录下的lib文件夹中。

[mysql驱动](http://central.maven.org/maven2/mysql/mysql-connector-java/5.1.38/mysql-connector-java-5.1.38.jar)

[dhcp依赖](http://central.maven.org/maven2/org/apache/commons/commons-dbcp2/2.1.1/commons-dbcp2-2.1.1.jar)

[commons-pool2依赖](http://maven.aliyun.com/nexus/service/local/artifact/maven/redirect?r=jcenter&g=org.apache.commons&a=commons-pool2&v=2.6.0&e=jar)

###二、主从备份

####1、修改jettty

&emsp;&emsp;首先修改conf->jetty.xml，这里是修改activemq的web管理端口，管理界面账号密码默认为admin/admin

```xml
<bean id="jettyPort" class="org.apache.activemq.web.WebConsolePort" init-method="start">
    <!-- the default port number for the web console -->
    <property name="port" value="8161"/>
</bean>
```

####2、修改activemq.xml

&emsp;&emsp;然后修改conf->activemq.xml

- 设置连接方式

  默认是下面五种连接方式都打开，这里我们只要tcp，把其他的都注释掉，然后在这里设置activemq的服务端口，可以看到每种连接方式都对应一个端口。

  ```xml
  <transportConnectors>
    <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
    <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <!-- <transportConnector name="amqp" uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="mqtt" uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="ws" uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/> -->
  </transportConnectors>
  ```

  

- 设置jdbc数据库

  mysql数据库中创建activemq库，在`broker`标签的下面也就是根标签`beans`的下一级创建一个bean节点，内容如下：

  ```xml
  <bean id="mysql-qs" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
      <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
      <property name="url" value="jdbc:mysql://localhost:3306/activemq?relaxAutoCommit=true"/>
      <property name="username" value="root"/>
      <property name="password" value="123456"/>
      <property name="poolPreparedStatements" value="true"/>
  </bean>
  ```

- 设置数据源

  首先修改broker节点，设置name和persistent(默认为true),也可不做修改，修改后如下：

  ```xml
  <broker xmlns="http://activemq.apache.org/schema/core" brokerName="mq1" persistent="true" dataDirectory="${activemq.data}">
  ```

  然后设置持久化方式,使用到我们之前设置的mysql-qs

  ```xml
  <persistenceAdapter>
    <!-- <kahaDB directory="${activemq.data}/kahadb"/> -->
    <jdbcPersistenceAdapter dataDirectory="${activemq.base}/activemq-data" dataSource="#mysql-qs"/>
  </persistenceAdapter>
  ```

#### 3、启动

&emsp;&emsp;设置完毕后启动activemq（双击bin中的acitveMQ.jar)，启动完成后可以看到如下日志信息：

```verilog
 INFO | Using a separate dataSource for locking: org.apache.commons.dbcp2.BasicDataSource@179ece50
 INFO | Attempting to acquire the exclusive lock to become the Master broker
 INFO | Becoming the master on dataSource: org.apache.commons.dbcp2.BasicDataSource@179ece50
```

​	接着我们修改一下tcp服务端口，改为61617，然后重新启动，日志信息如下：

```verilog
 INFO | Using a separate dataSource for locking: org.apache.commons.dbcp2.BasicDataSource@179ece50
 INFO | Attempting to acquire the exclusive lock to become the Master broker
 INFO | Failed to acquire lock.  Sleeping for 10000 milli(s) before trying again...
 INFO | Failed to acquire lock.  Sleeping for 10000 milli(s) before trying again...
 INFO | Failed to acquire lock.  Sleeping for 10000 milli(s) before trying again...
 INFO | Failed to acquire lock.  Sleeping for 10000 milli(s) before trying again...
```

可以看到从节点一直在尝试获取表锁成为主节点，这样一旦主节点失效，从节点能够立刻取代主节点提供服务。这样我们便实现了主从备份。

### 三、负载均衡

&emsp;&emsp;activemq可以实现多个mq之间进行路由，假设有两个mq，分别为brokerA和brokerB，当一条消息发送到brokerA的队列test中，有一个消费者连上了brokerB，并且想要获取test队列，brokerA中的test队列就会路由到brokerB上。

&emsp;&emsp;&emsp;开启负载均衡需要设置`networkConnectors`节点，静态路由配置如下：

```xml
<networkConnectors>
  <networkConnector uri="static:failover://(tcp://localhost:61616,tcp://localhost:61617)"           duplex="false"/>
</networkConnectors>
```

brokerA和brokerB都要设置该配置，以连上对方。

### 四、测试

####1、建立mq

&emsp;&emsp;组建两组broker，每组做主从配置。

- brokerA：
  - 主：设置web管理端口8761,设置mq名称`mq`,设置数据库地址为activemq，设置tcp服务端口61616，设置负载均衡静态路由`static:failover://(tcp://localhost:61618,tcp://localhost:61619)`,然后启动
  - 从：上面的基础上修改tcp服务端口为61617,然后启动
- brokerB:
  - 主：设置web管理端口8762，设置mq名称`mq1`,设置数据库地址activemq1，设置tcp服务端口61618，设置负载均衡静态路由`static:failover://(tcp://localhost:61616,tcp://localhost:61617)`,然后启动
  - 从：上面的基础上修改tcp服务端口为61619，然后启动

#### 2、springboot测试

&emsp;&emsp;&emsp;沿用上一篇的项目，修改配置文件的broker-url为`failover:(tcp://localhost:61616,tcp://localhost:61617,tcp://localhost:61618,tcp://localhost:61619)`，然后启动项目访问会在控制台看到如下日志：

```java
2018-07-31 15:09:25.076  INFO 12780 --- [ActiveMQ Task-1] o.a.a.t.failover.FailoverTransport       : Successfully connected to tcp://localhost:61618
1I'm from queue1:hello
2018-07-31 15:09:26.599  INFO 12780 --- [ActiveMQ Task-1] o.a.a.t.failover.FailoverTransport       : Successfully connected to tcp://localhost:61618
2I'm from queue1:hello
2018-07-31 15:09:29.002  INFO 12780 --- [ActiveMQ Task-1] o.a.a.t.failover.FailoverTransport       : Successfully connected to tcp://localhost:61616
1I'm from queue1:hello
2018-07-31 15:09:34.931  INFO 12780 --- [ActiveMQ Task-1] o.a.a.t.failover.FailoverTransport       : Successfully connected to tcp://localhost:61618
2I'm from queue1:hello
```

证明负载均衡成功。