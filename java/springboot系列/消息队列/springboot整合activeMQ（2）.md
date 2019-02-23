---
id: "2018-09-06-10-38"
date: "2018-09-06-10-38"
title: "springboot整合ActiveMQ（2）"
tags: ["java", "spring","springboot","消息队列","activeMQ"]
categories: 
- "java"
- "spring boot学习"
---

&emsp;&emsp;单个 MQ 节点总是不可靠的，一旦该节点出现故障，MQ 服务就不可用了，势必会产生较大的损失。这里记录 activeMQ 如何开启主从备份，一旦 master（主节点故障），slave（从节点）立即提供服务，实现原理是运行多个 MQ 使用同一个持久化数据源，这里以 jdbc 数据源为例。同一时间只有一个节点（节点 A）能够抢到数据库的表锁，其他节点进入阻塞状态，一旦 A 发生错误崩溃，其他节点就会重新获取表锁，获取到锁的节点成为 master，其他节点为 slave，如果节点 A 重新启动，也将成为 slave。

&emsp;&emsp;主从备份解决了单节点故障的问题，但是同一时间提供服务的只有一个 master，显然是不能面对数据量的增长，所以需要一种横向拓展的集群方式来解决面临的问题。

### 一、activeMQ 设置

#### 1、平台版本说明：

- 平台：windows
- activeMQ 版本：5.9.1，[下载地址](https://www.apache.org/dist/activemq/5.9.1/apache-activemq-5.9.1-bin.zip.asc)
- jdk 版本：1.8

#### 2、下载 jdbc 依赖

&emsp;&emsp;下载下面三个依赖包，放到 activeMQ 安装目录下的 lib 文件夹中。

[mysql 驱动](http://central.maven.org/maven2/mysql/mysql-connector-java/5.1.38/mysql-connector-java-5.1.38.jar)

[dhcp 依赖](http://central.maven.org/maven2/org/apache/commons/commons-dbcp2/2.1.1/commons-dbcp2-2.1.1.jar)

[commons-pool2 依赖](http://maven.aliyun.com/nexus/service/local/artifact/maven/redirect?r=jcenter&g=org.apache.commons&a=commons-pool2&v=2.6.0&e=jar)

###二、主从备份

####1、修改 jettty

&emsp;&emsp;首先修改 conf->jetty.xml，这里是修改 activemq 的 web 管理端口，管理界面账号密码默认为 admin/admin

```xml
<bean id="jettyPort" class="org.apache.activemq.web.WebConsolePort" init-method="start">
    <!-- the default port number for the web console -->
    <property name="port" value="8161"/>
</bean>
```

<!-- more -->

####2、修改 activemq.xml

&emsp;&emsp;然后修改 conf->activemq.xml

- 设置连接方式

  默认是下面五种连接方式都打开，这里我们只要 tcp，把其他的都注释掉，然后在这里设置 activemq 的服务端口，可以看到每种连接方式都对应一个端口。

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

* 设置 jdbc 数据库

  mysql 数据库中创建 activemq 库，在`broker`标签的下面也就是根标签`beans`的下一级创建一个 bean 节点，内容如下：

  ```xml
  <bean id="mysql-qs" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
      <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
      <property name="url" value="jdbc:mysql://localhost:3306/activemq?relaxAutoCommit=true"/>
      <property name="username" value="root"/>
      <property name="password" value="123456"/>
      <property name="poolPreparedStatements" value="true"/>
  </bean>
  ```

* 设置数据源

  首先修改 broker 节点，设置 name 和 persistent(默认为 true),也可不做修改，修改后如下：

  ```xml
  <broker xmlns="http://activemq.apache.org/schema/core" brokerName="mq1" persistent="true" dataDirectory="${activemq.data}">
  ```

  然后设置持久化方式,使用到我们之前设置的 mysql-qs

  ```xml
  <persistenceAdapter>
    <!-- <kahaDB directory="${activemq.data}/kahadb"/> -->
    <jdbcPersistenceAdapter dataDirectory="${activemq.base}/activemq-data" dataSource="#mysql-qs"/>
  </persistenceAdapter>
  ```

#### 3、启动

&emsp;&emsp;设置完毕后启动 activemq（双击 bin 中的 acitveMQ.jar)，启动完成后可以看到如下日志信息：

```verilog
 INFO | Using a separate dataSource for locking: org.apache.commons.dbcp2.BasicDataSource@179ece50
 INFO | Attempting to acquire the exclusive lock to become the Master broker
 INFO | Becoming the master on dataSource: org.apache.commons.dbcp2.BasicDataSource@179ece50
```

​ 接着我们修改一下 tcp 服务端口，改为 61617，然后重新启动，日志信息如下：

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

&emsp;&emsp;activemq 可以实现多个 mq 之间进行路由，假设有两个 mq，分别为 brokerA 和 brokerB，当一条消息发送到 brokerA 的队列 test 中，有一个消费者连上了 brokerB，并且想要获取 test 队列，brokerA 中的 test 队列就会路由到 brokerB 上。

&emsp;&emsp;&emsp;开启负载均衡需要设置`networkConnectors`节点，静态路由配置如下：

```xml
<networkConnectors>
  <networkConnector uri="static:failover://(tcp://localhost:61616,tcp://localhost:61617)"           duplex="false"/>
</networkConnectors>
```

brokerA 和 brokerB 都要设置该配置，以连上对方。

### 四、测试

####1、建立 mq

&emsp;&emsp;组建两组 broker，每组做主从配置。

- brokerA：
  - 主：设置 web 管理端口 8761,设置 mq 名称`mq`,设置数据库地址为 activemq，设置 tcp 服务端口 61616，设置负载均衡静态路由`static:failover://(tcp://localhost:61618,tcp://localhost:61619)`,然后启动
  - 从：上面的基础上修改 tcp 服务端口为 61617,然后启动
- brokerB:
  - 主：设置 web 管理端口 8762，设置 mq 名称`mq1`,设置数据库地址 activemq1，设置 tcp 服务端口 61618，设置负载均衡静态路由`static:failover://(tcp://localhost:61616,tcp://localhost:61617)`,然后启动
  - 从：上面的基础上修改 tcp 服务端口为 61619，然后启动

#### 2、springboot 测试

&emsp;&emsp;&emsp;沿用上一篇的项目，修改配置文件的 broker-url 为`failover:(tcp://localhost:61616,tcp://localhost:61617,tcp://localhost:61618,tcp://localhost:61619)`，然后启动项目访问会在控制台看到如下日志：

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
