## Servlet容器

什么Servlet容器，字节流发送给jvm之后，jvm里面有一个Servlet容器（常见的Tomcat/Jetty/JB oss），容器把字节流读取出来之后拼装成java内部对象HttpServletRequest(),Http协议在TCP上的字节流在java（jvm）内部的一个对象表示。Servlet容器帮我们把字节读取出来之后，封装成一个HttpServletRequset(ServletApi)

Servlet 

可以读取TCP端口上的字节流

按照HTTP协议进行解析

装配成符合Servlet标准的Java对象

实例化Servlet并按照标准调用其方法

![image-20200307110403519](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200307110403519.png)

Servlet做的就是把HTTP协议上裸的TCP字节流封装成一个 java对象。Spring中也有对Servlet的实现DispatcherServlet (用于把Servlet容器 发给我的请求分发给Spring容器内部处理的Servlet)。

## Spring容器

如果没有Spring两个程序互相依赖

```java
public class AService (){
    Bservice bservice;
}
public class BService (){
    AService aservice ;
}
public class Application {
    public static void main (String[] args){
        AService a = new AService ();
        BService b = new BService (); 
       	//到这还不能使用以为他们里面互相依赖的对象还没初始化
        a.bservice = b;
        a.aservice = a;
    }
}
```

没有Spring的时候，把他们创建出来然后需要梳理他们的依赖关系，把他们的依赖互相组装好。

####  Spring 中引入了DI和IOC

##### DI

声明好对象的依赖关系比如 a->b b->c a->c  spring发现了这依赖关系把他们注入进来 （反射）

#####  IOC 

原先需要控制每一个对象的依赖关系进行装配，现在把控制权交给容器本身容器读取依赖关系然后一个一个注入进去。

对比

自己手工创建对象，按照依赖关系装配对象。

声明对象的依赖关系，容器自动完成装配

 @Autowired/@Inject

##### 实现简单的IOC容器

https://github.com/zhangyang1995/simple-ioc-container/commit/4032f3959db3d79c205fdf56a51d08b8889ef332

```java
package com.github.hcsp.ioc;

import org.springframework.beans.factory.annotation.Autowired;

import java.io.IOException;
import java.lang.reflect.Field;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class MyIoCContainer {
    // 实现一个简单的IoC容器，使得：
    // 1. 从beans.properties里加载bean定义
    // 2. 自动扫描bean中的@Autowired注解并完成依赖注入
    // 先把所有的实体类放入到map中
    // 然后循环map 拿出每个bean实例 扫描bean实例中的字段Field 如果标注了@Autowired 则从bean中取出
    // 将指定对象变量上此 Field 对象表示的字段设置为指定的新值
    private static ConcurrentHashMap beans = new ConcurrentHashMap();

    public static void main(String[] args) {
        MyIoCContainer container = new MyIoCContainer();
        container.start();
        OrderService orderService = (OrderService) container.getBean("orderService");
        orderService.createOrder();
    }

    // 启动该容器
    public void start() {
        Properties properties = new Properties();
        try {
            properties.load(getClass().getResourceAsStream("/beans.properties"));
        } catch (IOException e) {
            e.printStackTrace();
        }
        properties.forEach((beanName, beanClass) -> {
            try {
                Class klass = Class.forName((String) beanClass);
                Object instance = klass.getConstructor().newInstance();
                beans.put(beanName, instance);
            } catch (Exception e) {
                e.getMessage();
            }
        });
        beans.forEach((beanName, beanInstance) -> di(beanInstance, beans));
    }

    public void di(Object instance, Map beans) {
        List<Field> fields = Stream.of(instance.getClass().getDeclaredFields()).filter((field) ->
                field.getAnnotation(Autowired.class) != null).collect(Collectors.toList());
        fields.forEach(field -> {
            String fieldName = field.getName();
            Object fieldInstance = beans.get(fieldName);
            field.setAccessible(true);
            try {
//                void set(Object obj, Object value)
//                将指定对象变量上此 Field 对象表示的字段设置为指定的新值。
                field.set(instance, fieldInstance);
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        });
    }

    // 从容器中获取一个bean
    public Object getBean(String beanName) {
        return beans.get(beanName);
    }
}

```

## Spring容器的启动

#####   前言

条件断点调试

一个 加载时候回去classpath中挨个找这个class文件 

如何读取一个资源

 ```java
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.started();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
     		//Spring图标
			Banner printedBanner = printBanner(environment);
            //创建applicationContext
			context = createApplicationContext();
			analyzers = new FailureAnalyzers(context);
            //准备Context
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
            //核心 AbstractApplicationContext
            //EmbeddedWebApplicationContext extends GenericWebApplicationContext
            //GenericWebApplicationContext extends GenericApplicationContext
            //GenericApplicationContext extends AbstractApplicationContext
            // AbstractApplicationContext 属于一个抽象模板 把加载bean的事情都做了
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			listeners.finished(context, null);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
 ```

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    //startupShutdownMonitor对象在spring环境刷新和销毁的时候都会用到，确保刷新和销毁不会同时执行
    synchronized (this.startupShutdownMonitor) {
        // 准备工作，例如记录事件，设置标志，检查环境变量等，并有留给子类扩展的位置，用来将属性加入到applicationContext中
        prepareRefresh();

        // 创建beanFactory，这个对象作为applicationContext的成员变量，可以被applicationContext拿来用,
        // 并且解析资源（例如xml文件），取得bean的定义，放在beanFactory中
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 对beanFactory做一些设置，例如类加载器、spel解析器、指定bean的某些类型的成员变量对应某些对象等
        prepareBeanFactory(beanFactory);

        try {
            // 子类扩展用，可以设置bean的后置处理器（bean在实例化之后这些后置处理器会执行）
            postProcessBeanFactory(beanFactory);

            // 执行beanFactory后置处理器（有别于bean后置处理器处理bean实例，beanFactory后置处理器处理bean定义）
            invokeBeanFactoryPostProcessors(beanFactory);

            // 将所有的bean的后置处理器排好序，但不会马上用，bean实例化之后会用到
            registerBeanPostProcessors(beanFactory);

            // 初始化国际化服务
            initMessageSource();

            // 创建事件广播器
            initApplicationEventMulticaster();

            // 空方法，留给子类自己实现的，在实例化bean之前做一些ApplicationContext相关的操作
            onRefresh();

            // 注册一部分特殊的事件监听器，剩下的只是准备好名字，留待bean实例化完成后再注册
            registerListeners();

            // 单例模式的bean的实例化、成员变量注入、初始化等工作都在此完成 重要
            finishBeanFactoryInitialization(beanFactory);

            // applicationContext刷新完成后的处理，例如生命周期监听器的回调，广播通知等
            finishRefresh();
        }

        catch (BeansException ex) {
            logger.warn("Exception encountered during context initialization - cancelling refresh attempt", ex);

            // 刷新失败后的处理，主要是将一些保存环境信息的集合做清理
            destroyBeans();

            // applicationContext是否已经激活的标志，设置为false
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }
    }
}
```

如何扫到注解 首先去 （classpath*:com/tenpay/bank/galaxy/channel/**/*.class） 里面获取.class文件

读入带有Component注解的class文件

```java
// Invoke factory processors registered as beans in the context.
invokeBeanFactoryPostProcessors(beanFactory);
```

![image-20200308215058824](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200308215058824.png)

![image-20200308221136134](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200308221136134.png)

将Bean放到容器中（此刻还没有实例化）![image-20200308222735354](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200308222735354.png)

![image-20200308224654037](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200308224654037.png)





##### 依赖注入

1.依赖field注入  通过一个beanNamePostProcessor  

```
AutowiredAnnotationBeanPostProcessor
```

![image-20200309205608718](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200309205608718.png)

2.构造器注入  

![image-20200309214134196](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200309214134196.png)

![image-20200309212519875](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200309212519875.png)

注意会造成循环引用

你有2个A B互相在构造器声明了对方会变成循环引用一个abean 和一个bbean在构造器里面都引用了对方你创造A必须先创造出来B，创造B先创造出来A 造成死锁

![image-20200309213027125](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200309213027125.png)

![image-20200309213110840](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200309213110840.png)

![image-20200309213152216](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200309213152216.png)

   改成Autowired Feild 可避免

因为Feild是在PostProcessor中，是等bean创建完在注入

而构造器注入是在你创建的时候就把相关依赖准备好，现在就要拿到依赖就会产生循环依赖

##### AOP

![image-20200309220935263](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200309220935263.png)

![image-20200309220528389](C:\Users\52377\AppData\Roaming\Typora\typora-user-images\image-20200309220528389.png)

 cglib 字节码增强 (在JVM里面生成了一个动态的类的字节码并且加载了它 classLoader defineClass() )

jdk动态代理  实现invocationHandler

共同点都生成了一个动态的代理对象，注入到了jvm内部，区别 jdk动态代理必须要求你实现一个借口 cglib 动态子类化生成一个子类化的.class

##### MVC

DispatchServlet（分发器） 实现了Servlet接口

分界线：servlet 和SpringMvc

![img](https://upload-images.jianshu.io/upload_images/17497804-d28c03332d41ce2f.png?imageMogr2/auto-orient/strip|imageView2/2/w/882/format/webp)