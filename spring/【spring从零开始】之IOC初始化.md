# Spring IOC容器

两种上下文初始化方式：

1、

spring boot 中使用的BeanFactory->ApplicationContext -> GenericApplicationContext->AnnotationConfigApplicationContext -> 使用java配置来实现将javabean信息注入到容器

2、

spring中最常用的BeanFactory->ApplicationContext ->ClassPathApplicationContext -> ClassPathXmlApplicationContext -> 使用xml来将javabean信息注入到容器

## ClassPathXmlApplicationContext 

最核心的逻辑是在ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();这一行中

```java
protected  ConfigurableListableBeanFactory  obtainFreshBeanFactory(){
{
    refreshBeanFactory();
  	return getBeanFactory();
}
```

 这里spring将会通过spring的xml文件生成一系列的javabean的包装bean -> BeanDefinition 然后放入默认生成的DefaultListableBeanFactory之中, 注意在spring中 所有的java bean在解析的过程中都会变成 BeanDefinition(有些特殊的单例不是用BeanDefinition封装的)

## AnnotationConfigApplicationContext

AnnotatedBeanDefinitionReader 和 ClassPathBeanDefinitionScanner 都是初始化用来将使用class或者basePackages名称加载的BeanDefinition注册进GenericApplicationContext的工具类

```java 
//初始化注册BeanDefinition所需要的加载器 -> 这两个方法很重要的 , 初始化了很多process比如AutowiredAnnotationBeanPostProcessor和
// CommonAnnotationBeanPostProcessor用来处理@Autowrite注解等等
public AnnotationConfigApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
}
//将指定的包下所有的配置类对象
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
		refresh();
}
//将指定的包下所有的配置类对象
public AnnotationConfigApplicationContext(String... basePackages) {
		this();
		scan(basePackages);
		refresh();
}
```

## IOC容器初始化

最核心的方法AbstractAplication的refresh()方法 , 这个方法完成了整个spring声明周期的初始化

```java 
@Override
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            //准备，记录容器的启动时间startupDate, 标记容器为激活，初始化上下文环境如文件路径信息，验证必填属性是否填写
            prepareRefresh();

            // 告诉子类刷新内部bean工厂,获取新的beanFactory，销毁原有beanFactory、为每个bean生成BeanDefinition等
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // 在这种情况下,bean工厂准备使用的.
            prepareBeanFactory(beanFactory);

            try {
                // 允许在上下文bean的后处理工厂子类。
                // 模板方法，此时，所有的beanDefinition已经加载，但是还没有实例化。允许在子类中对beanFactory进行扩展处理。比如添加ware相关接口自动装配设置，添加后置处理器等，是子类扩展prepareBeanFactory(beanFactory)的方法
                postProcessBeanFactory(beanFactory);
    		   //在上下文中调用factory工厂的时候注册bean的实例对象
                // 实例化和注册beanFactory中扩展了BeanPostProcessor的bean,
                //例如：
                // AutowiredAnnotationBeanPostProcessor(处理被@Autowired注解修饰的bean并注入)
                // RequiredAnnotationBeanPostProcessor(处理被@Required注解修饰的方法)
                
                invokeBeanFactoryPostProcessors(beanFactory);

                // 注册bean的过程当中拦截所以bean的创建
                registerBeanPostProcessors(beanFactory);

                // 初始化上下文消息资源
                initMessageSource();

                //初始化事物传播属性
                initApplicationEventMulticaster();

                // 在特定上下文初始化其他特殊bean子类。
                onRefresh();

                // 检查侦听器bean并注册。
                registerListeners();

                // 实例化所有剩余(non-lazy-init)单例.
                // 比如invokeBeanFactoryPostProcessors方法中根据各种注解解析出来的类，在这个时候都会被初始化。
                finishBeanFactoryInitialization(beanFactory);

                // 最后一步:发布对应的事件。
                // refresh做完之后需要做的其他事情。
                // 清除上下文资源缓存（如扫描中的ASM元数据）
                // 初始化上下文的生命周期处理器，并刷新（找出Spring容器中实现了Lifecycle接口的bean并执行start()方法）。
                // 发布ContextRefreshedEvent事件告知对应的ApplicationListener进行响应的操作
                finishRefresh();
            }

            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }

                // 销毁已经创建的单例对象避免浪费资源
                destroyBeans();

                // 重置“活跃”的旗帜。
                cancelRefresh(ex);

                // 异常传播到调用者。
                throw ex;
            }

            finally {
                // 在spring 核心包里重置了内存，因为我们肯不需要元数据单例bean对象了
                resetCommonCaches();
            }
        }
    }
```

以上完成IOC容器初始化和bean初始化，

# Bean的创建

`进入PostProcessorRegistrationDelegate类中的invokeBeanFactoryPostProcessors方法，在上下文中调用factory工厂的时候注册bean的 实例对象，`

`接下来进入AbstractBeanFactory.java类中的doGetBean方法，这个方法的具体实现可以分为三个部分：`

第一部分，首先先去singleton缓存中去找实例。由于我们例子中没有把我们的bean手动放入singletonObjects这个Map里面去，所以这里肯定没找到。

第二部分，然后是去获取该BeanFactory父Factory，希望从这些Factory中获取，如果该Beanfactory有父类，则希望用父类去实例化该bean，类似于JVM类加载的双亲委派机制。由于我们例子中的的Beanfactory为null，所以暂不讨论这种情况。

第三部分，这一部分是我们关注的重点，这里我们将这一大部分再分为三个小的部分来进行分析：

1. 先将目前的bean标记为的正在创建
2. 再获取根据beanName得到对应bean在beanfactory中的beanDefinitionMap的BeanDefinition（上一节初始化beanFactory时存入的），然后去获取这个bean依赖的bean。如果依赖的bean还没有创建，则先创建依赖的bean，进行递归调用（这就是依赖注入Dependence Injection）。如果找不到依赖，则忽略。
3. 最后如果是单例（Spring默认是单例），则调用createBean()这个方法进行Bean的创建。

`进入AbstractAutowireCapableBeanFactory.java类的createBean方法，这里面可以分为四个部分：`

第一部分：确保该bean的class是真实存在的，也就是该bean是可以classload可以找到加载的

第二部分：准备方法的重写

第三部分：可以看到，这边出现了一个return，也就是说这边可以返回bean了。但看注释：Give BeanPostProcessors a chance to return a proxy instead of the target bean instance. 这样就很清晰了，BeanPostProcessor这个接口是可以临时修改bean的，优先级高于正常实例化bean的，如果beanPostProcessor能返回，则直接返回了。

第四部分：调用doCreateBean方法开始对bean进行创建

`打开doCreateBean方法，在这个方法里会做两件事：一是通过createBeanInstance这个方法创建bean，二是通过initializeBean方法初始化bean。先看看createBeanInstance这个方法里有什么`

进入createBeanInstance方法，这块代码主要是再次对bean做安全检查并确定该bean有默认的构造函数。直接看这个方法最后一行，调用instantiateBean方法并返回方法的结果。

再进入SimpleInstantiationStrategy.java的instantiate方法，我们可以看到，在这个方法里，Spring通过反射的方法根据BeanDefinition创建出Bean的对象并返回。

# **初始化Bean**

让我们回到AbstractAutowireCapableBeanFactory.java类中的doCreateBean方法中，重点关注里面的initializeBean方法。现在bean已经被创建了，开始初始化该bean。

在这个方法中，先调用invokeAwareMethods方法用于加载相关资源（比如BeanName、BeanClassLoader、BeanFactory等资源）。

再调用applyBeanPostProcessorsBeforeInitialization方法用于构造方法执行之前再次修改Bean（BeanPostProcessor接口）。

然后通过invokeInitMethods调用自定义的初始化方法

再调用方法用于构造方法执行之前再次修改Bean（BeanPostProcessor接口）。

# 扩展点

```java
 1 当@Scope为singleton时,bean会在ioc初始化时就被实例化,默认为singleton,可以配合@Lazy实现延时加载
 2 当@Scope为prototype时,bean在ioc初始化时不会被实例化,只有在通过context.getBean()时才会被实例化
 3 执行顺序 Constructor > @PostConstruct > InitializingBean > init-method > SmartInitializingSingleton
 4 实现SmartInitializingSingleton接口的类被Spring实例化为一个单例bean,在所有的Bean加载完成后,才会被调用
 如果该类被设置为懒加载,那么SmartInitializingSingleton接口方法永远不会被触发,即使使用时bean被实例化了也不会触发
 5 其他的初始化方式不管是否懒加载,在对象被创建后都会被调用
 6 如果是通过成员变量注入依赖的对象,而不是通过构造函数注入,那么在调用构造方法时,成员变量是没有被注入的
 这也可以理解,因为只有有了对象之后才能通过代码对成员变量操作.
 (切记不是对象初始化,对象初始化之前是先初始化成员变量,不过这也是相对讲的,实际上实例化过程不仅仅这么简单)
 7 如果一个类被设置为懒加载,但是其他类注入该懒加载类,也会立刻实例化为Spring Bean.
 解决办法:可以在注入的地方也设置成懒加载
```