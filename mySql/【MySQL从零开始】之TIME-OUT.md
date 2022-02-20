> connect timeout：建立数据库连接超时
>
> socket timeout：socket读取超时
>
> statement timeout：单个sql执行超时
>
> transaction timeout：事务执行超时，一个事务中可能包含多个sql
>
> get connection timeout：从连接池中获取链接超时

## ** connectTimeout与socketTimeout**

connect timeout和socket timeout都属于TCP层面的超时。

以mysql为例，我们可以在jdbc url中指定connectTimeout和socketTimeout。如：

```
jdbc:mysql://localhost:3306/db?connectTimeout=1000&socketTimeout=60000

```

   其中：

> connectTimeout：表示的是数据库驱动(mysql-connector-java)与mysql服务器建立TCP连接的超时时间。
>
> socketTimeout：是通过TCP连接发送数据(在这里就是要执行的sql)后，等待响应的超时时间。
>
> mysql驱动(mysql-connector-java)在与服务端建立Socket连接时，会将这两个参数设置到socket对象上参见：
>
> com.mysql.jdbc.MysqlIO类的构造方法：
>
> 提示：这里的mysqlConnection类型为java.net.Socket  
>
> 如果这两个参数设置的不够合理，都会导致mysql驱动抛出以下异常：
>
> com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure  
>
> 相信大部分读者对这个异常都不陌生。接下来笔者将分别演示这两个异常是如何产生的，并提出对应的解决方案。

## **connectTimeout**

下面首先通过一个案例演示如何模拟connectTimeout

1. ```java
   1. @Test
   2. public void testConnectTimeout() throws SQLException {
   3.   DruidDataSource dataSource = new DruidDataSource();
   4.   dataSource.setInitialSize(5);
   5.   dataSource.setUrl("jdbc:mysql://127.0.0.1:3306/test?connectTimeout=5");
   6.   dataSource.setUsername("root");
   7.   dataSource.setPassword(“your password");
   8.   dataSource.setDriverClassName("com.mysql.jdbc.Driver”);
   9.   dataSource.init();*//初始化，底层通过mysql-connector-java建立数据库连接*
   10. }
   ```

   

笔者这里将connectTimeout设置为了5ms，表示mysql驱动与服务端建立一个连接最多不能超过5ms。由于这里是与本地(127.0.0.1)数据库建立一个连接，5ms已经足够。然而，如果你是与一个远程数据库建立连接，那么5ms可能无法完成建立一个连接，此时你极有可能会遇到类似以下异常：

1. ```java
   1. com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
   2. The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
   3.   at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
   4.   at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
   5.   ...
   6. Caused by: java.net.SocketTimeoutException: connect timed out
   7.   at java.net.PlainSocketImpl.socketConnect(Native Method)
   8.   at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
   9.   ...
   ```

   

2. 到这里，我们看到了：

CommunicationsException异常，异常的Caused by部分是

```java
java.net.SocketTimeoutException: connect timed out
```

​    也就是说，建立底层socket 连接超时了。这通常意味着我们需要将connectTimeout值调大。

​    这个问题并非无关紧要，特别是在公司有多个数据中心的情况下，尤其需要注意。笔者曾经遇到过有业务开发同学，应用部署在北京，数据库集群在北京和上海都有部署，如下图：

   

​    上海和北京的一个RTT大概在20ms，而业务同学将connectTimeout设置为10ms。这就是导致，应用与北京的主库建立连接可以成功，但是与上海的从库建立连接总是经常失败，显然问题的解决方案，就是调大connectTimeout的值。需要注意的是，通常建议connectTimeout设置的值是需要大于RTT的，如果设置的刚刚好，很容易因为网络拥堵或者抖动导致出现相同的异常。

​    最后，connectTimeout的默认值为0，驱动层面不设置超时时间，但这并不意味着不会超时。此时将由操作系统来决定超时时间。一些内核参数，如net.ipv4.tcp_syn_retries可以影响connectTimeout，这里不做深入介绍。

## ** socketTimeout**

​    socket timeout是我们实际开发中最容易遇到的另外一个导致CommunicationsException异常的原因，通常是在sql的执行时间超过了socket timeout设置的情况下出现。例如socket timeout设置的是3s，但是sql执行确需要5s，那么将会出现异常。

socket timeout异常演示：

```java
1. @Test
2.   public void testSocketTimeout() throws SQLException {
3.   org.apache.tomcat.jdbc.pool.DataSource datasource = new org.apache.tomcat.jdbc.pool.DataSource();
4.   *//设置socketTimeout=3000，单位是ms*
5.   datasource.setUrl("jdbc:mysql://localhost:3306/test?socketTimeout=3000");
6.   datasource.setUsername("root");
7.   datasource.setDriverClassName("com.mysql.jdbc.Driver");
8.   datasource.setPassword(“your password");
9.   Connection connection = datasource.getConnection();
10.   PreparedStatement ps = connection.prepareStatement("select sleep(5)");
11.   ps.executeQuery();
12. }
```



在这个案例中，我们模拟了一个慢查询，通过执行"select sleep(5)"，sleep是mysql提供的函数，其接受一个休眠时间，单位是s，当我们把这个sql发送给mysql时，mysql服务端会休眠5秒后，再返回结果。

然而，由于我们在jdbc url中设置了socketTimeout=3000，意味着单条sql最大执行时间不能超过3s。因此运行以上案例，将会抛出类似以下异常：

```java
1. com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
2. The last packet successfully received from the server was 3,080 milliseconds ago.  The last packet sent successfully to the server was 3,005 milliseconds ago.
3.   at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
4.   at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
5.   at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
6.   at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
7.   ...
8. Caused by: java.net.SocketTimeoutException: Read timed out
9.   at java.net.SocketInputStream.socketRead0(Native Method)
10.   at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
11.   ...
```

​    这个异常看起来与connectTimeout导致的异常很相似，但是实际却有很大不同。这里我们是执行了一条sql，Caused By部分的异常提示为Read timed out，而之前是建立连接时抛出的异常，异常提示为connect timeout。

在异常信息的开始部分，我们看到了详细的错误提示信息：最后一次接收到服务端返回的报文是3080ms之前，最后一次发送报文给服务端是3005ms之前。

​    细心的读者已经发现，3005ms与我们设置的socketTimeout=3000如此接近，事实上，你可以认为多出的5ms是系统检测到超过socketTimeout的耗时，之后抛出异常。当然，在实际开发中，系统检测socket timeout的耗时并不是固定为5ms，每次检测的耗时可能都不同，一般不过超过几十毫秒。

另外，socketTimeout是配置在jdbc url上的，对于所有执行的sql都会有这个超时限制。因此在配置这个值的时候，应该比应用中耗时最长的sql还要稍大一点。

socketTimeout默认值也是0，也就是不超时。

## **statement timeout**

socket timeout统一限制了所有SQL执行的最大耗时，有的时候，我们希望为不同的SQL指定不同的最大超时时间。这可以通过statement timeout来完成。

Statement对象提供了一个setQueryTimeout方法(其子类PreparedStatement继承了这个方法)，单位是秒，默认值为0，也就是 不超时。以下是一个设置statement timeout的案例：

```java
1. Connection conn = datasource.getConnection();
2. PreparedStatement ps = conn.prepareStatement("select sleep(5)");
3. ps.setQueryTimeout(1);//设置statement timeout
4. ps.executeQuery();
5. 在这里：
```

我们执行的sql是"select sleep(5)”，服务端需要休眠5s后才返回，

另外，我们设置了sql查询超时queryTimeout为1s

由于sql执行耗时超出了1s，因此，执行上述代码片段将抛出类似以下异常：

```java
1. com.mysql.jdbc.exceptions.MySQLTimeoutException: Statement cancelled due to timeout or client request
2.   at com.mysql.jdbc.PreparedStatement.executeInternal(PreparedStatement.java:1881)
3.   at com.mysql.jdbc.PreparedStatement.executeQuery(PreparedStatement.java:1962)
4.   at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
5.   at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
6.   at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
7.   at java.lang.reflect.Method.invoke(Method.java:498)
8.   at org.apache.tomcat.jdbc.pool.StatementFacade$StatementProxy.invoke(StatementFacade.java:114)
9.   at com.sun.proxy.$Proxy6.executeQuery(Unknown Source)
10.  
11.   ...
```



可以看到，提示的异常信息为"Statement cancelled due to timeout or client request"，表示sql由于执行超时而被取消了。

通过statement timeout，我们可以更加灵活的为不同的sql设置不同的超时时间。然而，在实际开发过程中，通常我们都是使用ORM框架，而不会直接使用原生的JDBC API，这意味着ORM要对此进行支持。

以mybatis为例，其提供了对statement timeout超时设置的支持。我们可以在<settings>元素中，为所有要执行的sql，设置一个默认的statement timeout。

如在mybatis-config.xml配置默认的statement timeout：

1. <settings>
2.  *<!--设置sql默认执行超时时间为25秒，如果为提供，则默认值为0，也就是没有限制-->* 
3.  <setting name="defaultStatementTimeout" value="25"/>
4. </settings>

或者在mapper映射文件中，指定单个sql的statement timeout，如

1. *<!--设置sql超时时间为10秒-->*
2. <select id="selectPerson" timeout="10" parameterType="int" resultType=“hashmap” >
3.  SELECT * FROM PERSON WHERE ID = #{id}
4. </select>

事实上，mybatis底层也是也只是我们我们配置的值，通过调用Statement.setQueryTimeout方法进行设置。

BaseStatementHandler*#setStatementTimeout*

需要注意的是，尽管statement timeout很灵活，但是在高并发的情况下，会创建大量的线程，一些场景下笔者并不建议使用。原因在于，mysql-connector-java底层是通过定时器Timer来实现statement timeout的功能，也就是说，对于设置了statement timeout的sql，将会导致mysql创建定时Timer来执行sql，意味着高并发的情况下，mysql驱动可能会创建大量线程。

以下是笔者模拟设置statement timeout之后，通过jstack命令查看的结果。

可以看到这里包含了一个名为Mysql Statement Cancellation Timer的线程，这就是用于控制sql执行超时的定时器线程。在高并发的情况下，大量的sql同时执行，如果设置了statement timeout，就会出现需要这样的线程。

在mysql-connector-java驱动的源码中(这里使用的是5.1.39版本)，体现了这个逻辑。在ConnectionImpl类中定义了一个超时Timer

com.mysql.jdbc.ConnectionImpl#getCancelTimer

这里我们看到ConnectionImpl内部，提供了一个名为MySQL Statement Cancellation Timer的定时器。

在sql执行时，如果设置了statement timeout，则将sql包装成一个task，通过Timer进行执行：mysql 驱动源码里有多处使用到了这个Timer，这里以StatementImpl的executeQuery方法为例进行讲解，包含了以下代码片段：

com.mysql.jdbc.StatementImpl#executeQuery

可以看到，在指定statement timeout的情况下，mysql内部会将sql执行操作包装成一个CancelTask，然后通过定时器Timer来运行。Timer实际上是与ConnectionImpl绑定的，同一个ConnectionImpl执行的多个sql，会共用这个Timer。默认情况下，这个Timer是不会创建的，一旦某个ConnectionImpl上执行的一个sql，指定了statement timeout，此时这个Timer才创建，一直到这个ConnectionImpl被销毁时，Timer才会取消。

在一些场景下，如分库分表、读写分离，如果使用的数据库中间件是基于smart-client方式实现的，会与很多库建立连接，由于其底层最终也是通过mysql-connector-java创建连接，这种场景下，如果指定了statement timeout，那么应用中将会存在大量的Timer线程，在这种场景下，并不建议设置。

最后，需要提醒的是，socket timeout是TCP层面的超时，是操作系统层面进行的控制，statement timeout是驱动层面实现的超时，是应用层面进行的控制，如果同时设置了二者，那么 socket timeout必须比statement timeout大，否则statement timeout无法生效。

## ** transaction timeout**

前面提到的的socket timeout、statement timeout，都是限制单个sql的最大执行超时。在事务的情况下，可能需要执行多个sql，我们想针对整个事务设置一个最大的超时时间。

例如，我们在采用spring配置事务管理器的时候，可以指定一个defaultTimeout属性，单位是秒，指定所有事务的默认超时时间。

  也可以在@Transactional注解上针对某个事务，指定超时时间，如：

```mysql
1. @Transactional(timeout = 3)
2. 如果同时配置了，@Transactional注解上的配置，将会覆盖默认的配置。
3.  
4. transaction timeout的实现原理可以用以下流程进行描述，假设事务超时为5秒，需要执行3个sql：
5.  
6.  start transaction  #事务超时为5秒
7.   |
8.  \|/
9.  sql1  #statement timeout设置为5秒
10.   |
11.   |   #执行耗时1s，那么整个事务超时还剩4秒 
12.  \|/
13.  sql2  #设置statement timeout设置为4秒
14.   |
15.   |   #执行耗时2秒，整个事务超时还是2秒
16.  \|/ 
17.  sql3  #设置statement timeout设置为2秒
18.   |
19.  ---  #假设执行耗时超过2s，那么整个事务超时，抛出异常  
```



这里只是一个简化的流程，但是可以帮助我们了解spring事务超时的原理。从这个流程中，我们可以看到，spring事务的超时机制，实际上是还是通过Statement.setQueryTimeout进行设置，每次都是把当前事务的剩余时间，设置到下一个要执行的sql中。

事实上，spring的事务超时机制，需要ORM框架进行支持，例如mybatis-spring提供了一个SpringManagedTransaction，里面有一个getTimeout方法，就是通过从spring中获取事务的剩余时间。这里不在继续进行源码分析。

## ** get connection timeout**

check connection timeout或者get connection timeout，表示从数据库连接池DataSource中获取链接超时。通DataSource的实现有很多，如druid，c3p0、dbcp2、tomcat-jdbc、hicaricp等，不同的连接池，抛出的异常类型不同，但是从异常的名字中，都可以看出是获取链接异常。连接池，底层也是通过mysql-connector-java创建连接，只不过在连接上做了一层代理，当关闭的时候，是返回连接池，而不是真正的关闭物理连接，从而达到连接复用。

 我们通常是需要首先获取到一个连接Connection对象，然后才能创建事务，设置事务超时实现，在事务中执行sql，设置sql的超时时间。因此，要操作数据库，Connection是基础。从连接池中，获取链接超时，是开发中，最常见的异常。

​    通常是因为连接池大小设置的不合理。如何设置合理的线程池大小需要进行综合考虑。

 这里以sql执行耗时、要支撑的qps为例：

​    假设某个接口的sql执行耗时为5ms，要支撑的最大qps为1000。一个sql执行5ms，理想情况下，一个Connection一秒可以执行200个sql。又因为支持的qps为1000，那么理论上我们只需要5个连接即可。当然，实际情况远远比这复杂，例如，我们没有考虑连接池内部的逻辑处理耗时，mysql负载较高执行sql变慢，应用发生了gc等，这些情况都会导致获取连接时间变长。所以，我的建议是，比理论值，高3-5倍。

**最后对以下两种典型情况，进行说明：**

​	1、应用启动时，出现获取连接超时异常

可以通过调大initPoolSize。如果连接池有延迟初始化(lazy init)功能，也要设置为立即初始化，否则，只有第一次请求访问数据库时，才会初始化连接池。这个时候容易出现获取链接超时。

2、业务高峰期，出现获取连接超时异常

如果是偶然出现，可以忽略。如果出现的较为频繁，可以考虑调大maxPoolSize和minPoolSize。

 