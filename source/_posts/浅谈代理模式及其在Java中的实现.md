---
title: 浅谈代理模式及其在Java中的实现
date: 2017-02-23 14:55:06
tags: ["Java", "设计模式"]
---
代理模式是常用的结构型设计模式之一，当无法直接访问某个对象或访问某个对象存在困难时可以通过一个代理对象来间接访问，为了保证客户端使用的透明性，所访问的真实对象与代理对象需要实现相同的接口。在Java中代理实现分为静态代理和动态代理，本文将简要描述代理模式及其在Java中的实现。
<!--more-->
## 代理模式
### 定义
>给某一个对象提供一个代理或占位符，并由代理对象来控制对原对象的访问。

### 结构
![代理模式UML图](http://upload-images.jianshu.io/upload_images/4778432-1465bad143ff3227.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*注意：上图的注释部分是针对Proxy类的request方法。*

由图可知，代理模式存在以下3种角色：

+ **Subject**:被代理类和代理类要实现的公共接口（*request()*)
+ **ConcreteSubject**:要被代理的类，又叫委托类，它定义了代理角色所代表的真实对象
+ **Proxy**:代理类，代理对象扮演着中介的角色，增加被代理类的某些功能或去掉了某些服务

### 简单实现
Subject接口：

```java
public interface Subject {
	void request();
}
```

ConcreteSubject类：
```java
public class ConcreteSubject implements Subject {
	@Override
	public void request() {
		System.out.println("=======具体子类的request=======");
	}
}
```

Proxy类：
```java
public class Proxy implements Subject {
	private ConcreteSubject subject;//被代理对象
	
	public Proxy(ConcreteSubject subject){
		this.subject=subject;
	}
	
	
	public void preRequest(){
		System.out.println("=======代理的preRequest=======");
	}
	
	@Override
	public void request() {
		preRequest();
		subject.request();//调用被代理对象的方法
		postRequest();
	}
	
	public void postRequest(){
		System.out.println("=======代理的postRequest=======");
	}

}
```

客户端测试类：
```java
public class Test {

	public static void main(String[] args) {
		//1.创建被代理类对象
		ConcreteSubject subject=new ConcreteSubject();
		//2.创建代理对象，使用构造注入把被代理类对象注入到代理对象中
		Proxy proxy=new Proxy(subject);
		//3.调用代理对象的相应方法
		proxy.request();
	}

}
```

运行测试类，应该看到如下结果:

![](http://upload-images.jianshu.io/upload_images/4778432-632b8a53db762103.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上，就实现了一个最简单的代理模式。接下来看一下常见的几种代理模式及其应用场景。

### 常见代理模式及应用场景
> 1. 远程代理(Remote Proxy)：为一个位于不同的地址空间的对象提供一个本地的代理对象，这
>个不同的地址空间可以是在同一台主机中，也可是在另一台主机中，远程代理又称为大使
>(Ambassador)。
>2. 虚拟代理(Virtual Proxy)：如果需要创建一个资源消耗较大的对象，先创建一个消耗相对较
>小的对象来表示，真实对象只在需要时才会被真正创建。
>3. 保护代理(Protect Proxy)：控制对一个对象的访问，可以给不同的用户提供不同级别的使用权限。
>4. 缓冲代理(Cache Proxy)：为某一个目标操作的结果提供临时的存储空间，以便多个客户端
>可以共享这些结果。
>5. 智能引用代理(Smart Reference Proxy)：当一个对象被引用时，提供一些额外的操作，例如将对象被调用的次数记录下来等。


其中，智能引用代理是使用得最广泛的一种代理，适合的应用如如：日志处理、权限管理、事务处理等。

## 代理的实现
代理模式的实现分为静态代理和动态代理。
### 静态代理
#### 概念

>由程序员创建或工具生成代理类的源码，再编译代理类。所谓静态也就是在程序运行前就已经存在代理类>的字节码文件，代理类和委托类的关系在运行前就确定了。Java编译完成后代理类是一个实际的 class 文件。

前面举的例子就是一个简单的静态代理。

静态代理的实现可分为继承和聚合，其中继承容易导致类膨胀，因此推荐的实现方式是聚合。

#### 静态代理优缺点
**优点：**业务类只需要关注业务逻辑本身，保证了业务类的重用性。这是代理的共有优点。

**缺点：**

1. 代理对象的一个接口只服务于一种类型的对象，如果要代理的方法很多，势必要为每一种方法都进行代理，静态代理在程序规模稍大时就无法胜任了。
2. 如果接口增加一个方法，除了所有实现类需要实现这个方法外，所有代理类也需要实现此方法。增加了代码维护的复杂度。

另外，如果要按照上述的方法使用代理模式，那么真实角色(委托类)必须是事先已经存在的，并将其作为代理对象的内部属性。但是实际使用时，一个真实角色必须对应一个代理角色，如果大量使用会导致类的急剧膨胀；此外，如果事先并不知道真实角色（委托类），该如何使用代理呢？这个问题可以通过Java的动态代理类来解决。

### 动态代理
#### 概念
>动态代理类的源码是在程序运行期间由JVM根据反射等机制动态的生成，所以不存在代理类的字节码文件。代理类和委托类的关系是在程序运行时确定。

#### 实现方式
Java中实现动态代理主要有两种典型方法：

1. 使用JDK实现动态代理
2. 使用[CGLIB](https://github.com/cglib/cglib)实现动态代理

我们主要来看一下第一种方法。

#### JDK实现动态代理
![JDK实现动态代理](http://upload-images.jianshu.io/upload_images/4778432-4e806d314eea261f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 步骤
1. 创建被代理的类及接口
2. 创建一个接口实现InvocationHandler的类(调用处理器)，它必须实现invoke方法
3. 调用Proxy的静态方法，动态创建一个代理类的对象
```java
new ProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)
```
4. 通过代理调用方法

##### 代码举例
看一个最简单的例子，大部分步骤已注释说明。

Subject接口：
```java
/**
 * 被代理类和代理类要实现的公共接口
 */
public interface Subject {
	void request();
}
```

要被代理的类：
```java
/**
 * Subject接口实现
 */
public class ConcreteSubject implements Subject{

	@Override
	public void request() {
		System.out.println("==========被代理对象的request方法==========");
	}
	
}
```

InvocationHandler实现类：
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;


/**
 * 代理类的调用处理器
 */
public class ProxyHandler implements InvocationHandler {
	//被代理对象
	private Subject target;
	
	public ProxyHandler(Subject target) {
		this.target=target;
	}
	
	/* 
	 * 参数：
	 * proxy:表示最终生成的代理类对象，注意不是被代理的对象
	 * method:被代理对象的方法
	 * args:方法的参数
	 * 返回值：
	 * Object方法的返回值
	 */
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		System.out.println("=========before===========");
		Object object=method.invoke(target);
		System.out.println("=========after===========");
		return object;
		
	}

}
```
测试类：
```java
import java.lang.reflect.Proxy;

public class ProxyTest {

	
	public static void main(String[] args) {
		//1.创建被代理对象
		ConcreteSubject subject=new ConcreteSubject();
		//2.创建调用处理器对象
		ProxyHandler proxyHandler=new ProxyHandler(subject);
		//3.动态生成代理对象
		Class<?> cls=ConcreteSubject.class;//获得被代理类的Class对象
		Subject proxySubject=(Subject)Proxy.newProxyInstance(cls.getClassLoader(), cls.getInterfaces(), proxyHandler);
		//4.通过代理对象调用方法
		proxySubject.request();
		
	}
}
```

运行测试类，可以得到以下结果：
![](http://upload-images.jianshu.io/upload_images/4778432-4cbd29050cf4b8fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此，用JDK实现了最简单的动态代理。
其中需要注意的地方是：

1. 需要创建一个java.lang.reflect.InvocationHandler的实现，作为代理类调用处理器。该接口只有一个方法：
```java
Object invoke(Object proxy, Method method, Object[] args)
```

	*注：参数说明见上文代码*

2. 调用
```java.lang.reflect.Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)```
获得代理实例

>参数说明：<br/>
>**loader:**定义代理类的类加载器<br/>
>**interfaces:**代理类需要实现的接口列表<br/>
>**h:**调用处理器，代理对象将会将方法调用转发给它<br/>
>返回值：<br/>
>**Object:**一个由上述参数所指定的代理实例

##### 动态代理实现思路
实现功能:通过Proxy的newProxyInstance返回代理对象

1. 声明一段源码（动态代理）
2. 编译源码（JDK Compiler API),产生新的类（代理类）
3. 将这个类load到内存当中，产生一个新的对象（代理对象）
4. return代理对象

注：可通过[慕课网：模拟JDK动态代理实现思路分析及简单实现](http://www.imooc.com/video/4903)了解
#### CGLIB
JDK提供的实现动态代理的方法十分强大，但是也有一定的限制，主要的限制是：
> 只能代理实现接口的类，没有实现接口的类不能实现JDK动态代理

可以使用CGLIB来解决该问题。CGLIB的特点是：

+ 针对类来实现代理
+ 对指定目标类生成一个子类，通过方法拦截技术拦截所有父类方法的调用
+ 由于使用继承实现，因此不能为final修饰的类或方法实现代理

#### 动态代理的应用
AOP(面向切面编程),即在不改变原有类方法的基础上，增加一些额外的业务逻辑。

Spring的AOP就是通过JDK动态代理跟CGLIB动态代理来实现的。

默认的策略是如果目标类是接口，则使用JDK动态代理技术，如果目标对象没有实现接口，则默认会采用CGLIB代理。

参考：

+ [Java代理和动态代理机制分析和应用](http://blog.csdn.net/fengyuzhengfan/article/details/49586277)
+ [慕课网：模式的秘密---代理模式](http://www.imooc.com/learn/214)
+ [C#设计模式之代理模式](http://blog.csdn.net/lovelion/article/details/8227953)