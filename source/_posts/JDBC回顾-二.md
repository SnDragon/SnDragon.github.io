---
title: JDBC回顾(二)
date: 2017-02-27 21:12:44
tags: ["Java", "JDBC"]
---
上一篇文章简单总结了JDBC的基本用法，这篇文章将继续介绍JDBC的更多用法，包括使用批处理，事务以及数据源等。

<!--more-->
## 批处理
### 简介
批处理主要运用于有大量SQL语句要执行的场景，如要执行10000条数据的插入，这是若一条一条的向数据库发送请求并执行，势必会造成效率低下。这时可以引入批处理来提升效率，当然，批处理的SQL语句条数会与硬件设备如内存容量等有关，因此要根据实际情况选择适当的批处理条数，以免导致内存溢出等问题。

### JDBC使用批处理
前文提到过除了调用存储过程之外，JDBC提供了Statement和PreparedStatement接口供程序员使用。下面分别介绍使用这两个接口来使用批处理
#### 使用Statement进行批处理
用到的API主要有：

>void addBatch(String sql) throws SQLException

将给定的sql语句添加到Statement对象当前的命令列表中，通过调用executeBatch()方法命令列表将被批量处理。

>void clearBatch() throws SQLException

清空Statement对象的当前SQL命令列表

>int[] executeBatch() throws SQLException

向数据库提交一批命令并请求执行，如果所有的命令都执行成功的话，返回一个更新数组成的数组。

---

下面是使用Statement进行SQL批处理的核心代码:

```java
	public static void testStatementBatch() throws SQLException {
		Connection conn=ConnectionUtil.getConnection();
		Statement stmt=conn.createStatement();
		String sql1="insert into account(user_name,balance) values('007',1000.0)";
		String sql2="update account set balance=balance+20 where balance=1000";
		stmt.addBatch(sql1);
		stmt.addBatch(sql2);
		stmt.executeBatch();
	}
```

查看数据库，应看到:


![](http://upload-images.jianshu.io/upload_images/4778432-5efd51c651571d67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 使用PreparedStatement进行批处理

直接看代码好了:
```java
    public static void testPreparedBatch() throws SQLException {
			Connection conn=ConnectionUtil.getConnection();
			String sql="insert into account(user_name,balance) values(?,?)";
			PreparedStatement pstmt=conn.prepareStatement(sql);
			long beginTime=System.currentTimeMillis();
			for(int i=0;i<1000;i++){
				pstmt.setString(1,"user"+(i+1));
				pstmt.setDouble(2, 1000.0);
				pstmt.addBatch();
				
				if((i+1)%100==0){
					pstmt.executeBatch();
					pstmt.clearBatch();
				}
			}
			pstmt.executeBatch();
			long endTime=System.currentTimeMillis();
			System.out.println("插入成功，共花了:"+(endTime-beginTime)+"ms");
	    }
```

运行结果：


![](http://upload-images.jianshu.io/upload_images/4778432-860c6a7947c9bd2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

数据库也插入了1000条记录。

接下来看一下不使用批处理的情况：
```java
    public static void testWithoutBatch() throws SQLException{
		Connection conn=ConnectionUtil.getConnection();
		String sql="insert into account(user_name,balance) values(?,?)";
		PreparedStatement pstmt=conn.prepareStatement(sql);
		long beginTime=System.currentTimeMillis();
		for(int i=0;i<1000;i++){
			pstmt.setString(1,"user"+(i+1));
			pstmt.setDouble(2, 1000.0);
			pstmt.executeUpdate();
		}
		long endTime=System.currentTimeMillis();
		System.out.println("插入成功，共花了:"+(endTime-beginTime)+"ms");
	}
```

结果：

![](http://upload-images.jianshu.io/upload_images/4778432-29b034a27b85b9e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*ps:同一台机器多次运行、不同机器运行都可能会出现不同的运行结果*

#### 两种实现批处理的方式对比
使用Statement.addBatch(sql)的优点是可以向数据库发送多条不同的SQL语句，主要的缺点是:SQL语句没有预编译，安全性跟运行速度都比较差。
我想PreparedStatement.addBatch()的优缺点你也知道了吧。


## 事务
不用说，事务自然是数据库中一个非常重要的概念。事务有ACID(原子性、一致性、隔离性、持久性)四个特性，想必很多地方都对这四个特性有详细的介绍，特别是经典的银行转账的例子还让人记忆犹新，因此此处就不再赘述。

今天我们要讲的是屌丝小明和他的女神之间的故事。先来复习一下触发器的使用吧。
### MySQL触发器
MySQL创建触发器的语法为:

```sql
	CREATE
	    [DEFINER = { user | CURRENT_USER }]
	    TRIGGER trigger_name
	    trigger_time trigger_event
	    ON tbl_name FOR EACH ROW
	    trigger_body
	
	trigger_time: { BEFORE | AFTER }
	
	trigger_event: { INSERT | UPDATE | DELETE }
```

我们假设向银行存钱的人太多了，银行钱放不下了因此银行规定每个用户的余额不能超过3000块(好吧，这是银行被黑得最惨的一次~)。根据该场景可以简单地创建如下触发器:

```sql
	DELIMITER //
	CREATE TRIGGER update_balance before update
	on account for each row
	BEGIN
	if new.balance>=3000 then
		update account set balance=old.balance where user_id=old.user_id;
	end if;
	END
	//
	DELIMITER ;
```

### JDBC与事务

这天，小明要给他的女神小红转1000块钱，先来看一下不用事务的代码：
```java
	public static void transferWithoutTransaction(int fromId,int toId,double money) throws SQLException {
			Connection conn=ConnectionUtil.getConnection();
			PreparedStatement pstmt=conn.prepareStatement("update account set balance=balance-? where user_id=?");
			pstmt.setDouble(1,money);
			pstmt.setInt(2, fromId);
			pstmt.executeUpdate();
			pstmt.setDouble(1, -money);
			pstmt.setInt(2, toId);
			pstmt.executeUpdate();
		}
```

*ps:为了简洁，省略异常处理和资源回收等代码*

回忆一下，之前小明，和小红的账户情况是：

![](http://upload-images.jianshu.io/upload_images/4778432-36aacf5959c0cd39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

执行

>transferWithoutTransaction(1,2,1000.0);

会抛出如下异常：


![](http://upload-images.jianshu.io/upload_images/4778432-6b267682ac35819c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是因为我们创建的触发器不允许更新账户余额为3000元及以上的原因。
看一下数据库：


![](http://upload-images.jianshu.io/upload_images/4778432-bb41e898132fbd0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看来小明要哭晕在厕所了。显然这是由于没有使用事务（默认是每句SQL语句执行后自动提交)的缘故，小明转账成功了，女神却没有收到钱。下面让我们来帮助小明摆脱可怜的命运吧,首先使用月光宝盒把数据改成原来的样子，然后完善我们的代码来使用合理的事务:

```java
	public static void transferWithTransaction(int fromId,int toId,double money){
		Connection conn=null;
		PreparedStatement pstmt=null;
		try{
			conn=ConnectionUtil.getConnection();
			conn.setAutoCommit(false);//取消自动提交
			pstmt=conn.prepareStatement("update account set balance=balance-? where user_id=?");
			pstmt.setDouble(1,money);
			pstmt.setInt(2, fromId);
			pstmt.executeUpdate();
			pstmt.setDouble(1, -money);
			pstmt.setInt(2, toId);
			pstmt.executeUpdate();
			
			conn.commit();
			System.out.println("事务提交成功!");
		}catch(SQLException e){
			System.out.println("出现异常，执行事务回滚!");
			try {
				conn.rollback();
			} catch (SQLException e1) {
				e1.printStackTrace();
			}
		}finally{
			try {
				if(pstmt!=null){
					pstmt.close();
				}
				if(conn!=null){
					pstmt.close();
				}
			} catch (SQLException e) {
				e.printStackTrace();
			}
		}
	}
```

执行

>transferWithTransaction(1,2,1000.0);

可以在控制台看到如下结果：


![](http://upload-images.jianshu.io/upload_images/4778432-8bd8802320148b3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再看下数据库：


![](http://upload-images.jianshu.io/upload_images/4778432-ca66c4bef076d4ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可见，尽管没有转账成功，小明的钱并没有不翼而飞，我们成功帮助了小明。

关键的地方在于关闭自动提交事务。
<pre>
conn.setAutoCommit(false);
</pre>
然后在没异常时提交(commit),在出现异常时回滚(rollback)。

事务是一个重要的概念，上面只是简单介绍了一个例子，并没有详细介绍事务，要了解更多，请自行查找关于事务与数据库的资料。

## 其他话题

### DataSource
javax.sql.DataSource是一个接口，主要作为DriverManager设施的替代项来获取Connection对象。为什么要替换DriverManager呢？

数据库连接的建立与关闭是非常耗费系统资源的操作，而使用传统的方式，即使用DriverManager获取的数据库连接，一个连接对象对应一个物理数据库连接，每次操作都打开一个物理连接，使用完立即关闭连接。如此频繁地打开、关闭连接势必会造成系统性能低下。

解决方案之一是使用数据库连接池：当应用程序启动时，系统主动建立足够的数据库连接，并将这些连接组成一个连接池。每次应用程序请求一个连接时，无须重新打开连接，而是从连接池中取出已有的连接使用，使用完后不立即关闭数据库连接，而是将连接返回给连接池。通过使用连接池，将大大提高程序的运行效率。Java多线程中的线程池也是遵循了类似的思想。

JDBC的数据库连接池使用DataSource来表示，具体实现通常由商用服务器或一些开源组织实现（如DBCP和C3P0等)。DataSource对象是获取连接的首选方法。实现 DataSource 接口的对象通常在基于 JavaTM Naming and Directory Interface (JNDI) API 的命名服务中注册。
 
DataSource有三种类型的实现： 

+ 基本实现——生成标准Connection对象
+ 连接池实现——生成自动参与连接池的Connection 对象。此实现与中间层连接池管理器一起使用。 
+ 分布式事务实现——生成一个Connection 对象，该对象可用于分布式事务，并且几乎始终参与连接池。此实现与中间层事务管理器一起使用，并且几乎始终与连接池管理器一起使用。DataSource对象的属性在需要时可以修改。例如，如果将数据源移动到另一个服务器，则可更改与服务器相关的属性。其优点是，因为可以更改数据源的属性，所以任何访问该数据源的代码都无需更改。 


在Spring与Hibernate、Mybatis等ORM框架的整合过程中，DataSource扮演着非常重要的角色。