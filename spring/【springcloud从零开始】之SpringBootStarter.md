![img](https://img-blog.csdnimg.cn/20200210223908535.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMzc2OTgz,size_16,color_FFFFFF,t_70)



springboot的项目启动时从main入口开始的：**所以我们从此开始分析启动原理和运行过程**

```java
@SpringBootApplication
public class SpringBoot05JpaApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBoot05JpaApplication.class, args);
	}
}
```

运行这个main方法，**主要由两个部分组成**：**new一个SpringApplication对象、运行该对象的run（args）方法。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725094312350.png)

## 一 SpringApplication对象

创建SpringApplication对象过程是：在构造中通过初始化来创建这个对象

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725094821221.png)

```java
@SuppressWarnings({ "unchecked", "rawtypes" })
	private void initialize(Object[] sources) {
	//（1）首先将SpringBoot05JpaApplication.class字节码对象判断，并保存下来，以后会用哪个。
		if (sources != null && sources.length > 0) {
			this.sources.addAll(Arrays.asList(sources));
		}
		//（2）判断整个项目是不是web环境
		this.webEnvironment = deduceWebEnvironment();
		//（3）实际从名字大概知道，从MATA-INF/spring.factories文件中获取ApplicationContextInitializer类型的所有类，
		//即保存起来
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
	   //（4）和第三步一样。调用的方法都一样，所以也是从从MATA-INF/spring.factories文件中获取
	   //ApplicationListener.class类型的所有类。用于后面的使用
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		//（5）最后一把是判断哪个配置类中有main方法入口，以此类为主配置类。
		this.mainApplicationClass = deduceMainApplicationClass();
	}

至此SpringApplication对象对象就创建完了
```

简述一下第三步中如何获取指定类型的类的简述过程：

（1）进入getSpringFactoriesInstances(ApplicationContextInitializer.class))方法汇总，**看内部都调用了那些方法**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725101614201.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)

**找到一个LoadFactoryNames（xxx）方法。进入继续看源码。找到一个常量，这个常量内容就是MATA-INF/spring.factories的路径**。**这就是初始化过程中自动加载指定类的过程。这一过程是springboot中所有自动配置的通用方式**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725101641234.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725101655296.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019072510094350.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)

至此，springboot中main方法启动的第一步就完成了——创建SpringApplication对象都做了些什么东东（初始化一些内容，保存一些listener，initializer等组件类）。这是所有springboot的通用步骤过程（无论集合什么技术栈这都是springboot应用的通用过程）

## 二 SpringApplication对象的run方法——最终返回完整的IOC容器

核心关注两点：（1）本项目生命周期和（2）IOC容器的生命周期。下面run方法是这两个生命周期的运行过程；实际上也可以通过两个类：SpringApplicationRunListener和ApplicationContextInitializer来监听他们生命周期过程

```java
public ConfigurableApplicationContext run(String... args) {
	//这些就是停止监听对象，然后后面启动它，这不用管
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		//这边创建IOC容器对象（核心注意这个对象奥！）。
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
		//这个和图形化 "java.awt.headless";有关，用的少，也忽略
		configureHeadlessProperty();
		//（1）这边就很重要了，获取本项目（应用的）中要运行的监听器对象。该对象如何获取的呢？下面有该对象获取过程
		//进入方法发现，还是从文件MATE-INFO/spring.factories中获取的。
		SpringApplicationRunListeners listeners = getRunListeners(args);
		//（2）启动该监听器对象；该监听器很重要奥！有对项目启动的监听，环境准备的监听，ioc容器的准备监听、IOC容器加载完成监听、项目run过程都结束后的监听。
		listeners.starting();
		try {
		//（3）这一句就是将参数对象化，将参数内容转为一个对象保存。下面也有该方法实现过程
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
		//（4）这一步是准备环境，各种IOC容器环境，监听器环境等等环境准备，而且还判断是否是web环境等等操作。有点复杂，
		//反正就知道这个方法准备好了各种环境就欧了
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
		//（5）从方法名就知道是打印标幅（图标）
			Banner printedBanner = printBanner(environment);
		//（6）这一步非常重要！！！，是创建spring的IOC容器
			context = createApplicationContext();
			//（7）这主要对容器进行分析的，如果有异常会有异常报告
			analyzers = new FailureAnalyzers(context);
			//（8）这一步比较关键IOC容器完成创建前的一些准备：如通用的监听器，环境等等都开始生效，为后面真正IOC所有内容生效作准备
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			//（9）这一步也很关键，是刷新初始化IOC容器中内容，就是一些自动配置，所有组件都生效的过程，这一步执行完后，IOC
			//容器也就基本彻底完成了。
			refreshContext(context);      //刷新容器就是给容器中初始化组件：所有的配置类，所有的bean都创建完了，如果web环境，内嵌的tomcat
			//也创建成功了。
			//（10）后面就是IOC容器创建后的一些善后工作和完善工作
			//如afterRefresh就是从ioc容器中获取所有的ApplicationRunner和CommandLineRunner进行回调 
			//ApplicationRunner先回调，CommandLineRunner再回调
			afterRefresh(context, applicationArguments);
			listeners.finished(context, null);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			//（11）最终返回启动的IOC容器，里面会有容器中注册的所有组件
			return context;
		}
				catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```

最后run方法返回的就是一个创建好的IOC容器，**这个容器已经经过初始化，创建，加载所有组件后的成型容器了。**
**（1）对应上面run方法中的序号语句（1）**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725110232176.png)

**（3）对应上面run方法中的序号语句（3）**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725110859800.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)

**（6）对应上面run方法中的序号语句（6）**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725113126802.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)

**（8）对应上面run方法中的序号语句（8）**
**这是给IOC创建内容前作最后的准备：如通用的监听器，环境等等都开始生效，为后面真正IOC所有内容生效作准备**

```java
private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
			//将环境内容保存到ioc容器中，然后进行环境设置。
		context.setEnvironment(environment);
		//给IOC容器后置处理
		postProcessApplicationContext(context);
		//该方法内部是调用最初创建SpringApplication对象时，保存的那些所有ApplicationContextInitializer类型（理解为初始化器）的初始化方法
		//终于知道前面保存那些类型的对象干嘛用了，是这边开始调用生效的。如下图所示
		applyInitializers(context);
		//这是监听器的初始操作
		listeners.contextPrepared(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		// Add boot specific singleton beans
	//bean工厂中注入那些参数对象，通过map对象，将对应的名和对象注册进工厂中
		context.getBeanFactory().registerSingleton("springApplicationArguments",
				applicationArguments);
		if (printedBanner != null) {
		//容器中获取bean工厂对象，通过map对象，将对应的名和对象添加进工厂中
			context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
		}
		// Load the sources  获取到主配置类，就是最main方法中run方法中的那个类
		Set<Object> sources = getSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[sources.size()]));
		listeners.contextLoaded(context);
	}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725141926929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)

（9）对应上面run方法中的序号语句（9）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725152633125.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)

## 其他相关分析

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725093640497.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)

上面的四个类中
（1）配置在META-INF/spring.factories然后容器到文件中自动加载
ApplicationContextInitializer、 SpringApplicationRunListener
（2） 只需要放在ioc容器中，然后容器对象调用：context.getBeansOfType(ApplicationRunner.class).values()
ApplicationRunner、CommandLineRunner

上面四个类都是在springboot运行过程中调用到的类，而且这些类我们都可以自定义，然后是springboot运行的时候，使用我们自定义的这几个类中方法。但是这四个类实现其接口就可以了，但是要配置到springboot项目中，才能被调用到，这几个类在上面分析源码时候发现，他们调用的方式不同，所以配置的方式也不同，如： ApplicationContextInitializer、 SpringApplicationRunListener这两个类调用是到META-INF/spring.factories下获取的，而ApplicationRunner、CommandLineRunner是直接容器工厂中获取的（context.getBeansOfType(ApplicationRunner.class).values()）。所以配置过程让项目调用的方式也不同。第一种要将自定义实现的类配置到源路径下的META-INF/spring.factories中，第二种直接在实现类上加个注解@component注册进容器就OK了。


```java
public interface ApplicationRunner {

	/**
	 * Callback used to run the bean.
	 * @param args incoming application arguments
	 * @throws Exception on error
	 */
	void run(ApplicationArguments args) throws Exception;

}


========================================
	private void callRunners(ApplicationContext context, ApplicationArguments args) {
		List<Object> runners = new ArrayList<Object>();
		runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
		runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
		AnnotationAwareOrderComparator.sort(runners);
		for (Object runner : new LinkedHashSet<Object>(runners)) {
			if (runner instanceof ApplicationRunner) {
				callRunner((ApplicationRunner) runner, args);
			}
			if (runner instanceof CommandLineRunner) {
				callRunner((CommandLineRunner) runner, args);
			}
		}
	}
===================================
		private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
		try {
			(runner).run(args);
		}
		catch (Exception ex) {
			throw new IllegalStateException("Failed to execute ApplicationRunner", ex);
		}
	}
```

上面发现这两个 ApplicationRunner、CommandLineRunner类是容器中获得对象调用的，所以我们可以自定义这个类型，然后注册进容器中，这样源码中调用该类型的话，就用我们自定义的了。

## 项目运行周期和IOC生命周期

这两个接口 ApplicationContextInitializer（容器初始化器）、 SpringApplicationRunListener（spring应用运行监听器），从名字就知道功能了。但值得注意的是，上面分析生命周期过程中，他们对象被调用的方式是从配置在META-INF/spring.factories中获取的，所以自定义这类后，必须依照源码中调用它的位置，而去配置我们自定义的类。

1.首先自定义 ApplicationContextInitializer（容器初始化器）

```java
package com.example.listener;

import org.springframework.context.ApplicationContextInitializer;
import org.springframework.context.ConfigurableApplicationContext;

public class myApplicationContextInitializer implements ApplicationContextInitializer
<ConfigurableApplicationContext> {

	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		String applicationName = applicationContext.getApplicationName();
		System.out.println("myApplicationContextInitializer........的初始化方法运行"+"........."
		+applicationName+"......");		
	}

}
```

#### 2.接着自定义SpringApplicationRunListener（spring应用运行监听器）

```java
package com.example.listener;

import java.util.Arrays;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.SpringApplicationRunListener;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.core.env.ConfigurableEnvironment;

public class MySpringApplicationRunListener implements SpringApplicationRunListener {

	public MySpringApplicationRunListener(SpringApplication application, String[] args) {
		// TODO Auto-generated constructor stub
	}
	
	@Override
	public void starting() {
		System.out.println("springboot应用运行开始。。。。。。");		
	}

	@Override
	public void environmentPrepared(ConfigurableEnvironment environment) {
		String[] activeProfiles = environment.getActiveProfiles();
		System.out.println(Arrays.toString(activeProfiles));
		System.out.println("springboot应用运行到应用的环境准备。。。。。。");		
	}

	@Override
	public void contextPrepared(ConfigurableApplicationContext context) {
		System.out.println("springboot应用运行到IOC对象创建前的准备。。。。。。");	
	}

	@Override
	public void contextLoaded(ConfigurableApplicationContext context) {
		System.out.println("springboot应用运行到IOC容器中内容加载前。。。。。。");	
		
	}

	@Override
	public void finished(ConfigurableApplicationContext context, Throwable exception) {
		System.out.println("springboot应用运行完美结束了。。。。。。");	
		
	}

}
```

### 3.将他们配置进指定调用的位置META-INF/spring.factories

这样就完成注册自定义的监听器的过程。这边监听器对象获取是从META-INF/spring.factories获取的，所以我们也要配置到这位置，但有些类是直接容器工厂中获取的（context.getBeansOfType(ApplicationRunner.class).values()）。这样我们自定义的类直接在上面@component注册进容器就OK了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725181234465.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)

最终通过监听器，确实可以了解到项目运行周期和IOC容器的生命周期，如下所示

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190725182627566.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bnhpbmcxMDAy,size_16,color_FFFFFF,t_70)
