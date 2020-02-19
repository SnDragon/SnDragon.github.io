---
title: JavaMail实现注册邮箱验证案例
date:  2017-03-04 17:01:05
tags: ["Java", "Mail"]
---
在日常生活中，我们在一个网站中注册一个账户时，往往在提交个人信息后，网站还要我们通过手机或邮件来验证，邮件的话大概会是下面这个样子的：

![](http://upload-images.jianshu.io/upload_images/4778432-142c77d9cebec478.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

用户通过点击链接从而完成注册，然后才能登录。

也许你会想，为什么要这么麻烦直接提交注册不就行了吗？这其中很大一部分原因是为了防止恶意注册。接下来让我们一起来使用最简单的JSP+Servlet的方式来完成一个通过邮箱验证注册的小案例吧。
<!--more-->
## 准备工作
### 前提知识
动手实践之前，你最好对以下知识有所了解：

+ JSP和Servlet
+ Maven
+ MySQL
+ [c3p0](http://www.mchange.com/projects/c3p0/)
+ [SMTP协议](https://zh.wikipedia.org/zh-hans/%E7%AE%80%E5%8D%95%E9%82%AE%E4%BB%B6%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)和[POP3协议](https://zh.wikipedia.org/wiki/%E9%83%B5%E5%B1%80%E5%8D%94%E5%AE%9A)

如果对邮件收发过程完全不了解的话，可以花三分钟的时间到[慕课网](http://www.imooc.com/video/14260)了解一下，讲得算是非常清楚了，这里就不赘述了。放张图回忆一下：

![邮件收发过程](http://upload-images.jianshu.io/upload_images/4778432-fbe6de795b6c0468.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 邮箱准备
在了解的上述内容之后，要实现这个案例，首先我们还得有两个邮箱账号，一个用来发送邮件，一个用来接收邮件。本案例使用QQ邮箱向163邮箱发送激活邮件，因此需要登录QQ邮箱，在设置->账户面板中开启POP3/SMTP服务，以允许我们通过第三方客户端发送邮件：

![](http://upload-images.jianshu.io/upload_images/4778432-25bf4413c22a9d64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


还要注意的是，登录以下服务: POP3/IMAP/SMTP/Exchange/CardDAV/CalDAV服务时，需要用到授权码而不是QQ密码，授权码是用于登录第三方邮件客户端的专用密码。因此我们需要获得授权码，以在后面的程序中使用。

好了，到此准备工作就差不多了，下面开始动手吧。

## 实现注册Demo
### 创建Maven工程
本次案例基于Maven,因此你要先创建一个Maven的Web工程，并引入相关依赖：
```xml
	<dependencies>
			<!-- JavaEE依赖 -->
			<dependency>
				<groupId>javaee</groupId>
				<artifactId>javaee-api</artifactId>
				<version>5</version>
				<scope>test</scope>
			</dependency>
			<dependency>
				<groupId>taglibs</groupId>
				<artifactId>standard</artifactId>
				<version>1.1.2</version>
			</dependency>
			<!-- mysql驱动依赖 -->
			<dependency>
				<groupId>mysql</groupId>
				<artifactId>mysql-connector-java</artifactId>
				<version>5.1.40</version>
			</dependency>
			<!-- c3p0依赖 -->
			<dependency>
				<groupId>c3p0</groupId>
				<artifactId>c3p0</artifactId>
				<version>0.9.1.2</version>
			</dependency>
			<!-- JavaMail相关依赖 -->
			<dependency>
				<groupId>javax.mail</groupId>
				<artifactId>mail</artifactId>
				<version>1.4.7</version>
			</dependency>
			<dependency>
				<groupId>javax.activation</groupId>
				<artifactId>activation</artifactId>
				<version>1.1.1</version>
			</dependency>
	
	</dependencies>
```

### 创建数据库表
接下来使用MySQL创建一张简单的用户表：
```sql
	create table `user`(
		id int(11) primary key auto_increment comment '用户id',
	    username varchar(255) not null comment '用户名',
	    email varchar(255) not null comment '用户邮箱',
	    password varchar(255) not null comment '用户密码',
	    state int(1) not null default 0 comment '用户激活状态：0表示未激活，1表示激活',
	    code varchar(255) not null comment '激活码'
	)engine=InnoDB default charset=utf8;
```

其中要注意的地方是state字段（用来判断用户账号是否激活）和code字段（激活码）。
### 创建注册页面
使用JSP创建一个最简单的注册页面(请自行忽略界面):

![](http://upload-images.jianshu.io/upload_images/4778432-340ef0a21e4a44c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

嗯，果然够简单。

### 主要的业务逻辑

先想一下，我们的整个流程应该是这样的：

1. 用户填写相关信息，点击注册按钮
2. 系统先将用户记录保存到数据库中，其中用户状态为未激活
3. 系统发送一封邮件并通知用户去验证
4. 用户登录邮箱并点击激活链接
5. 系统将用户状态更改为已激活并通知用户注册成功

搞清楚了整个流程，实现起来应该就不难了。下图是我建立的包结构：

![](http://upload-images.jianshu.io/upload_images/4778432-8604f4b8ad70052a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


*ps:完整代码请见后文链接，这里只讨论主要的思路*

首先是，用户提交注册信息后，相应的servlet会将相关信息传给service层去处理，在service中需要做的就是讲记录保存到数据库中(调用dao层),然后再给用户发送一封邮件，UserServiceImpl相关代码如下:
```java
	public boolean doRegister(String userName, String password, String email) {
		// 这里可以验证各字段是否为空
		
		//利用正则表达式（可改进）验证邮箱是否符合邮箱的格式
		if(!email.matches("^\\w+@(\\w+\\.)+\\w+$")){
			return false;
		}
		//生成激活码
		String code=CodeUtil.generateUniqueCode();
		User user=new User(userName,email,password,0,code);
		//将用户保存到数据库
		UserDao userDao=new UserDaoImpl();
		//保存成功则通过线程的方式给用户发送一封邮件
		if(userDao.save(user)>0){
			new Thread(new MailUtil(email, code)).start();;
			return true;
		}
		return false;
	}
```
需要注意的是，应该新建一个线程去执行发送邮件的任务，不然被骂估计是免不了了。
数据库的操作比较简单，此处就不贴出来了，无非是将用户记录插到数据库中。值得一提的是，此处使用c3p0来作为数据源来替代DriverManager，在频繁获取释放数据库连接时效率会大大提高，c3p0最简单的配置如下：
```xml
	<?xml version="1.0" encoding="UTF-8"?>
	<c3p0-config>
		<named-config name="mysql">
			<property name="driverClass">com.mysql.jdbc.Driver</property>
			<property name="jdbcUrl">jdbc:mysql://127.0.0.1:3306/test1?useSSL=false</property>
			<property name="user">root</property>
			<property name="password">123456</property>
			<!-- 初始化时一个连接池尝试获得的连接数量，默认是3，大小应该在maxPoolSize和minPoolSize之间 -->
			<property name="initialPoolSize">5</property>
			<!-- 一个连接最大空闲时间(单位是秒)，0意味着连接不会过时 -->
			<property name="maxIdleTime">30</property>
			<!-- 任何指定时间的最大连接数量 ,默认值是15 -->
			<property name="maxPoolSize">20</property>
			<!-- 任何指定时间的最小连接数量 ,默认值是3 -->
			<property name="minPoolSize">5</property>
		</named-config>
	</c3p0-config> 
```

提供一个工具类DBUtil以获取，释放连接：
	
```java
 public class DBUtil {
	private static ComboPooledDataSource cpds=null;
	
	static{
		cpds=new ComboPooledDataSource("mysql");
	}
	
	public static Connection getConnection(){
		Connection connection=null;
		try {
			connection = cpds.getConnection();
		} catch (SQLException e) {
			e.printStackTrace();
		}
		return connection;
	}
	
	public static void close(Connection conn,PreparedStatement pstmt,ResultSet rs){
		try {
			if(rs!=null){
				rs.close();
			}
			if(pstmt!=null){
				pstmt.close();
			}
			if(rs!=null){
				rs.close();
			}
		} catch (SQLException e) {
			e.printStackTrace();
		}
		
	}
}
```

要特别注意的一点是：即使是使用连接池，使用完Connection后调用close方法，当然这不意味着关闭与数据库的TCP 连接，而是将连接还回到池中去，如果不close掉的话，这个连接将会一直被占用，直到连接池中的连接耗尽为止。


### 使用JavaMail发送邮件
使用JavaMail发送邮件非常简单，也是三步曲：

1. 创建连接对象javax.mail.Session
2. 创建邮件对象 javax.mail.Message
3. 发送邮件

直接看代码，详细的注释在代码中，MailUtil代码如下:
```java


	public class MailUtil implements Runnable {
		private String email;// 收件人邮箱
		private String code;// 激活码
	
		public MailUtil(String email, String code) {
			this.email = email;
			this.code = code;
		}
	
		public void run() {
			// 1.创建连接对象javax.mail.Session
			// 2.创建邮件对象 javax.mail.Message
			// 3.发送一封激活邮件
			String from = "xxx@qq.com";// 发件人电子邮箱
			String host = "smtp.qq.com"; // 指定发送邮件的主机smtp.qq.com(QQ)|smtp.163.com(网易)
	
			Properties properties = System.getProperties();// 获取系统属性
	
			properties.setProperty("mail.smtp.host", host);// 设置邮件服务器
			properties.setProperty("mail.smtp.auth", "true");// 打开认证
	
			try {
				//QQ邮箱需要下面这段代码，163邮箱不需要
				MailSSLSocketFactory sf = new MailSSLSocketFactory();
				sf.setTrustAllHosts(true);
				properties.put("mail.smtp.ssl.enable", "true");
				properties.put("mail.smtp.ssl.socketFactory", sf);


				// 1.获取默认session对象
				Session session = Session.getDefaultInstance(properties, new Authenticator() {
					public PasswordAuthentication getPasswordAuthentication() {
						return new PasswordAuthentication("xxx@qq.com", "xxx"); // 发件人邮箱账号、授权码
					}
				});
	
				// 2.创建邮件对象
				Message message = new MimeMessage(session);
				// 2.1设置发件人
				message.setFrom(new InternetAddress(from));
				// 2.2设置接收人
				message.addRecipient(Message.RecipientType.TO, new InternetAddress(email));
				// 2.3设置邮件主题
				message.setSubject("账号激活");
				// 2.4设置邮件内容
				String content = "<html><head></head><body><h1>这是一封激活邮件,激活请点击以下链接</h1><h3><a href='http://localhost:8080/RegisterDemo/ActiveServlet?code="
						+ code + "'>http://localhost:8080/RegisterDemo/ActiveServlet?code=" + code
						+ "</href></h3></body></html>";
				message.setContent(content, "text/html;charset=UTF-8");
				// 3.发送邮件
				Transport.send(message);
				System.out.println("邮件成功发送!");
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}
```

*ps:需要把上面的账号、授权码进行相应修改。*

完成后，再有用户提交注册信息时，应该就能收到验证邮件了：

![](http://upload-images.jianshu.io/upload_images/4778432-3ccc4e47dd4c90e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

用户点击链接后，我们要做的工作就是根据code（可以利用UUID生成)更改数据库中相应用户的状态，然后提示用户注册结果了。

## 总结
简单介绍了如何使用JavaMail完成了一个带邮箱验证的注册案例，当然在实际开发中还有许多细节要注意，例如对用户提交信息的校验，密码进行加密等，此处的简单案例并未详尽处理这些细节。
### 代码
[RegisterDemo](https://github.com/SnDragon/JavaLearning/tree/master/RegisterDemo)