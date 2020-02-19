---
title: JDBC回顾(一)
date: 2017-02-27 17:024:53
tags: ["Java", "JDBC"]
---
Java数据库连接，（Java Database Connectivity，简称JDBC）是Java语言中用来规范客户端程序如何来访问数据库的应用程序接口，提供了诸如查询和更新数据库中数据的方法。JDBC是面向关系型数据库的，同时JDBC也是Hibernate、Mybatis等框架的基础。本文主要回顾下JDBC的基本知识。
<!--more-->
## JDBC使用步骤
我们都知道，使用JDBC不外乎遵循下面几个步骤：

1. 加载特定的数据库驱动
2. 获得连接对象
3. 创建SQL语句
4. 执行SQL语句（增、删、改、查）
5. 拿到数据库中数据并进行相应的处理
6. 关闭相关的资源

## 准备活动
要复习JDBC，离不开数据库，这里首先要用MySQL创建一张表跟插入几条数据：

```sql
	create database jdbc;
	use jdbc;
	#创建账户表
	create table account(
		user_id int primary key auto_increment comment '用户id',
	    user_name varchar(20) not null comment '用户名',
	    balance double not null default 0.0 comment '用户余额'
	    );
	#插入几条数据
	insert into account(user_name,balance) values('小明',1000.0),('小红',2000.0),('小李',1500.0);
```

## 三种语句
JDBC在java.sql包中提供了三个语句接口供程序员使用，分别是Statement、PreparedStatement以及CallableStatement它们之间的类图如下：

![](http://upload-images.jianshu.io/upload_images/4778432-8a7bed4a478948bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

即PreparedStatement接口继承了Statement接口，而CallableStatement接口又继承了PreparedStatement接口。至于三种接口有什么区别，请看下文。

### Statement
Statement用于执行一条静态的SQL语句并返回它执行的结果。
下面来看一个最简单的JDBC例子：

```java
	public class TestStatement {
	
		public static void main(String[] args) {
			final String URL="jdbc:mysql://localhost:3306/jdbc?useUnicode=true&characterEncoding=utf-8&useSSL=false";
			final String USER="root";
			final String PASSWORD="123456";
			Connection conn=null;
			Statement stmt=null;
			ResultSet rs=null;
			
			try {
				//加载数据库驱动程序
				Class.forName("com.mysql.jdbc.Driver");
				//获得连接对象
				conn=DriverManager.getConnection(URL,USER,PASSWORD);
				//创建语句对象
				stmt=conn.createStatement();
				//执行一条查询的SQL语句,并获取结果集
				rs=stmt.executeQuery("select * from account");
				//对结果集进行处理
				while(rs.next()){
					int id=rs.getInt(1);
					String name=rs.getString("user_name");
					double balance=rs.getDouble("balance");
					System.out.println("id:"+id+" name:"+name+"  balance:"+balance);
				}
				//构造并执行一条插入的SQL语句
				String sql ="insert into account(user_name,balance) values('小王',500)";
				stmt.executeUpdate(sql);
			} catch (ClassNotFoundException e) {
				e.printStackTrace();//可以用日志文件记录下来
			} catch (SQLException e) {
				e.printStackTrace();
			}finally{
				//关闭相关的资源
				try{
					if(rs!=null){
						rs.close();
						rs=null;
					}
					if(stmt!=null){
						stmt.close();
						stmt=null;
					}
					if(conn!=null){
						conn.close();
						conn=null;
					}
				}catch(Exception e){
					e.printStackTrace();
				}
			}
		}
	}
```

运行上述代码，不出错的话会看到下列结果：

![](http://upload-images.jianshu.io/upload_images/4778432-e45a1a64ec4102b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

查看数据库，可以看到小王那条记录也成功插入了：

![](http://upload-images.jianshu.io/upload_images/4778432-059a2f4ffb3ada2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*ps:不想在数据库工具与IDE来回切换的话可以在IDE(如eclipse,IDEA等)中进行相应配置，请自行搜索*

然而，大部分我们不会直接使用Statement,而是使用运行速度更快且安全性更高的PreparedStatement。

### PreparedStatement
PreparedStatement对象代表着一条预编译好的语句，因此相对于Statement执行效率更高。
下面看一个PreparedStatement的例子：

```java
	public class TestPreparedStatement {
		public static void main(String[] args) {
			final String URL="jdbc:mysql://localhost:3306/jdbc?useUnicode=true&characterEncoding=utf-8&useSSL=false";
			final String USER="root";
			final String PASSWORD="123456";
			Connection conn=null;
			PreparedStatement pstmt=null;
			ResultSet rs=null;
			String s ="insert into account(user_name,balance) values(?,?)";
			try {
				Class.forName("com.mysql.jdbc.Driver");
				conn=DriverManager.getConnection(URL,USER,PASSWORD);
				pstmt=conn.prepareStatement(s);
				pstmt.setString(1,"小吴");
				pstmt.setDouble(2, 1000.0);
				int number=pstmt.executeUpdate();
				if(number>0){
					System.out.println("插入记录成功!");
				}
			}
			//...
		}
	}
```

*ps:为节省空间，省去catch语句块和finally语句块的内容*

查看数据库，应该可以看到记录插入成功。

以下是PreparedStatement和Statement的主要区别：

+ PreparedStatement可以写动态参数化的查询(使用占位符"?")
+ 由于使用了预编译，PreparedStatement的运行速度更快
+ PreparedStatement能过滤掉特殊字符(如单引号等),因此能有效防止[SQL注入](https://zh.wikipedia.org/wiki/SQL%E8%B3%87%E6%96%99%E9%9A%B1%E7%A2%BC%E6%94%BB%E6%93%8A)等攻击，安全性更高

从上面几点来看，我们都应该尽可能地使用PreparedStatement。然而，如果要调用存储过程，还得用到PreparedStatement的子接口CallableStatement。
### CallableStatement
#### 简介
>存储过程（Stored Procedure）是在大型数据库系统中，一组为了完成特定功能的SQL语句集，存储在数据库中，经过第一次编译后再次调用不需要再次编译，用户通过指定存储过程的名字并给出参数（如果该存储过程带有参数）来执行它。

可以通过CallableStatement用来调用存储过程。先来看一下MySQL创建存储过程的语法：

```sql
	CREATE
	    [DEFINER = { user | CURRENT_USER }]
	    PROCEDURE sp_name ([proc_parameter[,...]])
	    [characteristic ...] routine_body
	
	proc_parameter:
	    [ IN | OUT | INOUT ] param_name type
	
	characteristic:
	    COMMENT 'string'
	  | LANGUAGE SQL
	  | [NOT] DETERMINISTIC
	  | { CONTAINS SQL | NO SQL | READS SQL DATA | MODIFIES SQL DATA }
	  | SQL SECURITY { DEFINER | INVOKER }
	
	routine_body:
	    Valid SQL routine statement
```

下面使用MySQL创建三个存储过程，分别不带参数，带输出参数和带输入参数：

```sql
	#无参数存储过程(查询所有记录)
	DELIMITER //
	CREATE PROCEDURE `select_nofilter`()
	BEGIN
		SELECT * from account;
	END
	//
	DELIMITER ;
	
	#带输出参数的存储过程(查询记录条数)
	DELIMITER //
	CREATE PROCEDURE `select_count`(out count int)
	BEGIN
		SELECT COUNT(*) into count from account;
	END
	//
	DELIMITER ;
	
	#带输入参数的存储过程(查询余额大于money的记录)
	DELIMITER //
	CREATE PROCEDURE `select_filter`(in money double)
	BEGIN
		select * from account where balance > money;
	END
	//
	DELIMITER ;
```

在MySQL中调用存储过程可使用call命令，如：
```sql
call select_filter(1000);
```

查看存储过程有两种方法：

+ `	select `name` from mysql.proc where db = 'your_db_name' and `type` = 'PROCEDURE'`
+ `show procedure status;`


查看存储过程创建：

```sql
show create procedure proc_name;
```

#### 调用
接下来，使用JDBC调用存储过程。
为了方便，先创建一个Connection工具类：

```java
	public class ConnectionUtil {
		private final static String URL="jdbc:mysql://localhost:3306/jdbc?useUnicode=true&characterEncoding=utf-8";
		private final static String USER="root";
		private final static String PASSWORD="123456";
		
		private static Connection conn=null;
		
		static{
			try {
				Class.forName("com.mysql.jdbc.Driver");
			} catch (ClassNotFoundException e) {
				e.printStackTrace();
			}
		}
		
		public static Connection getConnection(){
			if(conn!=null){
				return conn;
			}
			
			try {
				conn=DriverManager.getConnection(URL,USER,PASSWORD);
			} catch (SQLException e) {
				e.printStackTrace();
			}
			return conn;
		}
	}
```

##### 不带参数的存储过程
调用不带参数的存储过程核心代码：

```java
	public static void callProcedureWithoutP() throws SQLException {
		Connection conn=ConnectionUtil.getConnection();
		CallableStatement cstmt=conn.prepareCall("call select_nofilter()");
		cstmt.execute();
		ResultSet rs=cstmt.getResultSet();
		while(rs.next()){
			System.out.println("name:"+rs.getString("user_name")+" balance:"+rs.getString("balance"));
		}
	}
```

结果：


![](http://upload-images.jianshu.io/upload_images/4778432-3a672871262cd604.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 带输入参数的存储过程
调用带输入参数的存储过程核心代码：

```java
	public static void callProcedureWithIn(double money) throws SQLException {
		Connection conn=ConnectionUtil.getConnection();
		CallableStatement cstmt=conn.prepareCall("call select_filter(?)");
		cstmt.setDouble(1, money);
		cstmt.execute();
		ResultSet rs=cstmt.getResultSet();
		while(rs.next()){
			System.out.println("name:"+rs.getString("user_name")+" balance:"+rs.getString("balance"));
		}
	}
```

结果：

![](http://upload-images.jianshu.io/upload_images/4778432-f9955b83021c5b7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 带输出参数的存储过程
调用带输出参数的存储过程核心代码：

```java
	public static void callProcedureWithOut() throws SQLException {
		Connection conn=ConnectionUtil.getConnection();
		CallableStatement cstmt=conn.prepareCall("call select_count(?)");
		cstmt.registerOutParameter(1, Types.INTEGER);
		cstmt.execute();
		int rowNumber=cstmt.getInt(1);
		System.out.println("rowNumber:"+rowNumber);
	}
```

结果：

![](http://upload-images.jianshu.io/upload_images/4778432-4551d02e7418ca74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结
以上是JDBC的基本用法，包括主要包括JDBC编程步骤，三种Statement使用方法和一些MySQL的语法。为了避免篇幅过长，将更多用法如：批处理、事务和数据源等放在下一篇文章中。
### 参考
[维基百科](https://zh.wikipedia.org/wiki/Java%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%9E%E6%8E%A5)