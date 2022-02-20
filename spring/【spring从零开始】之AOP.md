# 概述：

AOP实现方式有很多种，包括反射、元数据处理、程序处理、拦截器处理等

**AOP不一定都像Spring AOP那样，是在运行时生成代理对象来织入的，还可以在编译期、类加载期织入，比如AspectJ**

## AOP简介：

- Aspect（切面）： Aspect 声明类似于 Java 中的类声明，在 Aspect 中会包含着一些 Pointcut 以及相应的 Advice。
- Joint point（连接点）：表示在程序中明确定义的点，典型的包括方法调用，对类成员的访问以及异常处理程序块的执行等等，它自身还可以嵌套其它 joint point。所有的方法(joint point) 都可以织入 Advice, 但是我们并不希望在所有方法上都织入 Advice, 而 Pointcut 的作用就是提供一组规则来匹配joinpoint, 给满足规则的 joinpoint 添加 Advice
- Pointcut（切点）：表示一组 joint point，这些 joint point 或是通过逻辑关系组合起来，或是通过通配、正则表达式等方式集中起来，它定义了相应的 Advice 将要发生的地方。
- Advice（增强）：Advice 定义了在 Pointcut 里面定义的程序点具体要做的操作，它通过 before、after 和 around 来区别是在每个 joint point 之前、之后还是代替执行的代码。
- Target（目标对象）：织入 Advice 的目标对象.。
- Weaving（织入）：将 Aspect 和其他对象连接起来, 并创建 Adviced object 的过程

## AOP自定义标签初始化

### 时序图

![img](https://img-blog.csdn.net/20180428154933900?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5bDYwMTk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



### 流程说明

1）AOP标签的定义解析刘彻骨肯定是从NamespaceHandlerSupport的实现类开始解析的，这个实现类就是AopNamespaceHandler。至于为什么会是从NamespaceHandlerSupport的实现类开始解析的，这个的话我想读者可以去在回去看看Spring自定义标签的解析流程，里面说的比较详细。

2）要启用AOP，我们一般会在Spring里面配置<aop:aspectj-autoproxy/>  ，所以在配置文件中在遇到aspectj-autoproxy标签的时候我们会采用AspectJAutoProxyBeanDefinitionParser解析器

3）进入AspectJAutoProxyBeanDefinitionParser解析器后，调用AspectJAutoProxyBeanDefinitionParser已覆盖BeanDefinitionParser的parser方法，然后parser方法把请求转交给了AopNamespaceUtils的registerAspectJAnnotationAutoProxyCreatorIfNecessary去处理

4）进入AopNamespaceUtils的registerAspectJAnnotationAutoProxyCreatorIfNecessary方法后，先调用AopConfigUtils的registerAspectJAnnotationAutoProxyCreatorIfNecessary方法，里面在转发调用给registerOrEscalateApcAsRequired，注册或者升级AnnotationAwareAspectJAutoProxyCreator类。对于AOP的实现，基本是靠AnnotationAwareAspectJAutoProxyCreator去完成的，它可以根据@point注解定义的切点来代理相匹配的bean。

5）AopConfigUtils的registerAspectJAnnotationAutoProxyCreatorIfNecessary方法处理完成之后，接下来会调用useClassProxyingIfNecessary() 处理proxy-target-class以及expose-proxy属性。如果将proxy-target-class设置为true的话，那么会强制使用CGLIB代理，否则使用jdk动态代理，expose-proxy属性是为了解决有时候目标对象内部的自我调用无法实现切面增强。

6）最后的调用registerComponentIfNecessary 方法，注册组建并且通知便于监听器做进一步处理。

## AOP代理对象初始化

   **1： 创建`AnnotationAwareAspectJAutoProxyCreator`对象
   2： 扫描容器中的切面，创建`PointcutAdvisor`对象
   3： 生成代理类**

AOP的核心逻辑是在AnnotationAwareAspectJAutoProxyCreator类里面实现，上层接口实现了BeanPostProcessor接口,

重写postProcessBeforeInstantiation，其主要目的在于如果用户使用了自定义的TargetSource对象，则直接使用该对象生成目标对象，而不会使用Spring的默认逻辑生成目标对象，并且这里会判断各个切面逻辑是否可以应用到当前bean上，如果可以，则直接应用，也就是说TargetSource为使用者在Aop中提供了一个自定义生成目标bean逻辑的方式，并且会应用相应的切面逻辑。对于postProcessAfterInstantiation方法，其主要作用在于Spring生成某个bean之后，将相关的切面逻辑应用到该bean上

### 时序图

![img](https://img-blog.csdn.net/2018042815551721?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5bDYwMTk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 流程说明

### postProcessBeforeInstantiation

bean实例化之后, 初始化之前操作postProcessBeforeInstantiation

在AbstractAutoProxyCreator类中实现BeanPostProcessor中的下面方法中:

```java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		Object cacheKey = getCacheKey(beanClass, beanName);
 
		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			//advisedBeans用于存储不可代理的bean，如果包含直接返回
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			//判断当前bean是否可以被代理，然后存入advisedBeans
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}
 
		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
		//获取封装当前bean的TargetSource对象，如果不存在，则直接退出当前方法，否则从TargetSource
         // 中获取当前bean对象，并且判断是否需要将切面逻辑应用在当前bean上。
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
              // 获取能够应用当前bean的切面逻辑
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
 
             // 对生成的代理对象进行缓存
			this.proxyTypes.put(cacheKey, proxy.getClass());
			//如果最终可以获得代理类，则返回代理类，直接执行实例化后置通知方法
			return proxy;
		}
 
		return null;
	}
```

### postProcessAfterInitialization

bean初始完成之后创建代理对象过程:postProcessAfterInitialization

在AbstractAutoProxyCreator类中实现BeanPostProcessor中的下面方法中

```java

public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			//缓存键：1.beanName不为空的话，使用beanName（FactoryBean会在见面加上"&"）
			//2.如果beanName为空，使用Class对象作为缓存的key
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				//如果条件符合，则为bean生成代理对象
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

###  **wrapIfNecessary**

  1.如果已经处理过，且该bean没有被代理过，则直接返回该bean
  2.如果该bean是内部基础设置类Class 或 配置了该bean不需要代理，则直接返回bean（返回前标记该bean已被处理过）
  3.获取所有适合该bean的增强Advisor
  如果增强不为null，则为该bean创建代理对象，并返回结果
  标记该bean已经被处理过

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		//如果已经处理过（targetSourcedBeans存放已经增强过的bean）
		if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		//advisedBeans的key为cacheKey，value为boolean类型，表示是否进行过代理
		//已经处理过的bean,不需要再次进行处理，节省时间
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		//是否是内部基础设置类Class || 配置了指定bean不需要代理,如果是的话，直接缓存。
    	//如果一个bean继承自Advice、Pointcut、Advisor、AopInfrastructureBean 或者 带有@Aspect注解，或被Ajc（AspectJ编译器）编译都会被认定为内部基础设置类
    	//在AnnotationUtils类中的findAnnotation方法中,判断这个bean上的注解类型是不是@Aspect
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}
 
		// 获取当前对象所有适用的Advisor.加入当前对象是orderController,那么找到所有切点是他的对应的@Aspect注解的类
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		//如果获取的增强不为null，则为该bean创建代理（DO_NOT_PROXY=null）
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
             //创建代理对象时候会用到是否进行JDK代理或者CGLIB代理
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}
		//标记该cacheKey已经被处理过
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```



AspectJAwareAdvisorAutoProxyCreator的实现**wrapIfNecessary方法中判断是否要进行****代理的方法**getAdvicesAndAdvisorsForBean同时会调用

findEligibleAdvisors处理两件事：

- findCandidateAdvisors找到Spring中所有的Advisor.
- findAdvisorsThatCanApply过滤出适合当前对象的advisors

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
         
        //找到Spring中Advisor的实现类（findCandidateAdvisors）
        //将所有拥有@Aspect注解的类转换为advisors（aspectJAdvisorsBuilder.buildAspectJAdvisors）
        List<Advisor> candidateAdvisors = findCandidateAdvisors();
 
       /* findAdvisorsThatCanApply
        找到当前对象适合的所有Advisor。整个过程比较简单：
        遍历所有的advisor。
         查看当前advisor的pointCut是否适用于当前对象，如果是，进入候选队列，否则跳过。*/
 
        List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
        //添加一个默认的advisor，执行时用到。
        extendAdvisors(eligibleAdvisors);
        if (!eligibleAdvisors.isEmpty()) {
            eligibleAdvisors = sortAdvisors(eligibleAdvisors);
        }
        return eligibleAdvisors;
    }
```

`findCandidateAdvisors()`方法最终调用的是`BeanFactoryAdvisorRetrievalHelper.findAdvisorBeans()`方法，我们首先看看该方法的实现：

```java

public List<Advisor> findAdvisorBeans() {
    String[] advisorNames = null;
    synchronized (this) {
        advisorNames = this.cachedAdvisorBeanNames;
        if (advisorNames == null) {
            // 获取当前BeanFactory中所有实现了Advisor接口的bean的名称
            advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                this.beanFactory, Advisor.class, true, false);
            this.cachedAdvisorBeanNames = advisorNames;
        }
    }
    if (advisorNames.length == 0) {
        return new LinkedList<>();
    }
 
    // 对获取到的实现Advisor接口的bean的名称进行遍历
    List<Advisor> advisors = new LinkedList<>();
    for (String name : advisorNames) {
        // isEligibleBean()是提供的一个hook方法，用于子类对Advisor进行过滤，这里默认返回值都是true
        if (isEligibleBean(name)) {
            // 如果当前bean还在创建过程中，则略过，其创建完成之后会为其判断是否需要织入切面逻辑
            if (this.beanFactory.isCurrentlyInCreation(name)) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Skipping currently created advisor '" + name + "'");
                }
            } else {
                try {
                    // 将当前bean添加到结果中
                    advisors.add(this.beanFactory.getBean(name, Advisor.class));
                } catch (BeanCreationException ex) {
                    // 对获取过程中产生的异常进行封装
                    Throwable rootCause = ex.getMostSpecificCause();
                    if (rootCause instanceof BeanCurrentlyInCreationException) {
                        BeanCreationException bce = (BeanCreationException) rootCause;
                        String bceBeanName = bce.getBeanName();
                        if (bceBeanName != null && 
                            this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                            if (logger.isDebugEnabled()) {
                                logger.debug("Skipping advisor '" + name + 
                                    "' with dependency on currently created bean: " 
                                    + ex.getMessage());
                            }
                            continue;
                        }
                    }
                    throw ex;
                }
            }
        }
    }
    return advisors;
}
//其中的 buildAspectJAdvisors方法，会触发ReflectiveAspectJAdvisorFactory中的getAdvisors方法：
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
		//从 aspectMetadata 中获取 Aspect()标注的类 class对象
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		//获取Aspect()标注的类名
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
		validate(aspectClass);
 
		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);
 
		List<Advisor> advisors = new LinkedList<>();
		//遍历该类所有方法，根据方法判断是否能获取到对应 pointCut，如果有，则生成 advisor 对象
		for (Method method : getAdvisorMethods(aspectClass)) {
                        //这里继续看下面的解析
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}
 
		// If it's a per target aspect, emit the dummy instantiating aspect.
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}
 
		// Find introduction fields.
		//获取 @DeclareParents 注解修饰的属性（并不常用）
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}
 
		return advisors;
	}

	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
			int declarationOrderInAspect, String aspectName) {
 
		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
		//根据候选方法名，来获取对应的 pointCut
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}
		//如果能获取到 pointCut，则将切点表达式 expressionPointcut、当前
		对象ReflectiveAspectJAdvisorFactory、 方法名等包装成 advisor 对象
		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}
//InstantiationModelAwarePointcutAdvisorImpl的构造方法会触发构造通知对象：

public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
		//......
		//根据注解类型，匹配对应的通知类型
		switch (aspectJAnnotation.getAnnotationType()) {
			//前置通知
			case AtBefore:
				springAdvice = new AspectJMethodBeforeAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			//最终通知
			case AtAfter:
				springAdvice = new AspectJAfterAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			//后置通知
			case AtAfterReturning:
				springAdvice = new AspectJAfterReturningAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
					springAdvice.setReturningName(afterReturningAnnotation.returning());
				}
				break;
			//异常通知
			case AtAfterThrowing:
				springAdvice = new AspectJAfterThrowingAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
				}
				break;
			//环绕通知
			case AtAround:
				springAdvice = new AspectJAroundAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			//切面
			case AtPointcut:
				if (logger.isDebugEnabled()) {
					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
				}
				return null;
			default:
				throw new UnsupportedOperationException(
						"Unsupported advice type on method: " + candidateAdviceMethod);
		}
 
		//......
	}
 
```

获取到当前bean的增强方法后，便调用createProxy方法，创建代理。先创建代理工厂proxyFactory，然后获取当前bean 的增强器advisors，把当前获取到的增强器添加到代理工厂proxyFactory，然后设置当前的代理工的代理目标对象为当前bean，最后根据配置创建JDK的动态代理工厂，或者CGLIB的动态代理工厂，然后返回proxyFactory

有两种主要的方式JDK的动态代理和CGLIB的动态代理，接下来，我们先来看看AOP动态代理的实现选择方式，先上核心实现代码

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {  
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {  
        Class targetClass = config.getTargetClass();  
        if (targetClass == null) {  
            throw new AopConfigException("TargetSource cannot determine target class: " +  
                    "Either an interface or a target is required for proxy creation.");  
        }  
        if (targetClass.isInterface()) {  
            return new JdkDynamicAopProxy(config);  
        }  
        return CglibProxyFactory.createCglibProxy(config);  
    }  
    else {  
        return new JdkDynamicAopProxy(config);  
    }  
} 
```

### 实现AOP代理

#### Spring JDK动态代理实现

```java
//各个步骤执行的说明：
//1）获取拦截器
//2）判断拦截器链是否为空，如果是空的话直接调用切点方法
//3）如果拦截器不为空的话那么便创建ReflectiveMethodInvocation类，把拦截器方法都封装在里面，也就是执行 //getInterceptorsAndDynamicInterceptionAdvice方法
public Object invoke(Object proxy, Method method, Object[] args) throwsThrowable {  
       MethodInvocation invocation = null;  
       Object oldProxy = null;  
       boolean setProxyContext = false;  
   
       TargetSource targetSource = this.advised.targetSource;  
       Class targetClass = null;  
       Object target = null;  
   
       try {  
           //eqauls()方法，具目标对象未实现此方法  
           if (!this.equalsDefined && AopUtils.isEqualsMethod(method)){  
                return (equals(args[0])? Boolean.TRUE : Boolean.FALSE);  
           }  
   
           //hashCode()方法，具目标对象未实现此方法  
           if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)){  
                return newInteger(hashCode());  
           }  
   
           //Advised接口或者其父接口中定义的方法,直接反射调用,不应用通知  
           if (!this.advised.opaque &&method.getDeclaringClass().isInterface()  
                    &&method.getDeclaringClass().isAssignableFrom(Advised.class)) {  
                // Service invocations onProxyConfig with the proxy config...  
                return AopUtils.invokeJoinpointUsingReflection(this.advised,method, args);  
           }  
   
           Object retVal = null;  
   
           if (this.advised.exposeProxy) {  
                // Make invocation available ifnecessary.  
                oldProxy = AopContext.setCurrentProxy(proxy);  
                setProxyContext = true;  
           }  
   
           //获得目标对象的类  
           target = targetSource.getTarget();  
           if (target != null) {  
                targetClass = target.getClass();  
           }  
   
           //获取可以应用到此方法上的Interceptor列表  
           List chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method,targetClass);  
   
           //如果没有可以应用到此方法的通知(Interceptor)，此直接反射调用 method.invoke(target, args)  
           if (chain.isEmpty()) {  
                retVal = AopUtils.invokeJoinpointUsingReflection(target,method, args);  
           } else {  
                //创建MethodInvocation  
                invocation = newReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);  
                retVal = invocation.proceed();  
           }  
   
           // Massage return value if necessary.  
           if (retVal != null && retVal == target &&method.getReturnType().isInstance(proxy)  
                    &&!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {  
                // Special case: it returned"this" and the return type of the method  
                // is type-compatible. Notethat we can't help if the target sets  
                // a reference to itself inanother returned object.  
                retVal = proxy;  
           }  
           return retVal;  
       } finally {  
           if (target != null && !targetSource.isStatic()) {  
                // Must have come fromTargetSource.  
               targetSource.releaseTarget(target);  
           }  
           if (setProxyContext) {  
                // Restore old proxy.  
                AopContext.setCurrentProxy(oldProxy);  
           }  
       }  
    }  
```



```java
//4)其实实际的获取工作其实是由AdvisorChainFactory. getInterceptorsAndDynamicInterceptionAdvice()这个方法来完成的，获取到的结果会被缓存，下面来分析下这个方法的实现：
/** 
    * 从提供的配置实例config中获取advisor列表,遍历处理这些advisor.如果是IntroductionAdvisor, 
    * 则判断此Advisor能否应用到目标类targetClass上.如果是PointcutAdvisor,则判断 
    * 此Advisor能否应用到目标方法method上.将满足条件的Advisor通过AdvisorAdaptor转化成Interceptor列表返回. 
    */  
    publicList getInterceptorsAndDynamicInterceptionAdvice(Advised config, Methodmethod, Class targetClass) {  
       // This is somewhat tricky... we have to process introductions first,  
       // but we need to preserve order in the ultimate list.  
       List interceptorList = new ArrayList(config.getAdvisors().length);  
   
       //查看是否包含IntroductionAdvisor  
       boolean hasIntroductions = hasMatchingIntroductions(config,targetClass);  
   
       //这里实际上注册一系列AdvisorAdapter,用于将Advisor转化成MethodInterceptor  
       AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();  
   
       Advisor[] advisors = config.getAdvisors();  
        for (int i = 0; i <advisors.length; i++) {  
           Advisor advisor = advisors[i];  
           if (advisor instanceof PointcutAdvisor) {  
                // Add it conditionally.  
                PointcutAdvisor pointcutAdvisor= (PointcutAdvisor) advisor;  
                if(config.isPreFiltered() ||pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {  
                    //TODO: 这个地方这两个方法的位置可以互换下  
                    //将Advisor转化成Interceptor  
                    MethodInterceptor[]interceptors = registry.getInterceptors(advisor);  
   
                    //检查当前advisor的pointcut是否可以匹配当前方法  
                    MethodMatcher mm =pointcutAdvisor.getPointcut().getMethodMatcher();  
   
                    if (MethodMatchers.matches(mm,method, targetClass, hasIntroductions)) {  
                        if(mm.isRuntime()) {  
                            // Creating a newobject instance in the getInterceptors() method  
                            // isn't a problemas we normally cache created chains.  
                            for (intj = 0; j < interceptors.length; j++) {  
                               interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptors[j],mm));  
                            }  
                        } else {  
                            interceptorList.addAll(Arrays.asList(interceptors));  
                        }  
                    }  
                }  
           } else if (advisor instanceof IntroductionAdvisor){  
                IntroductionAdvisor ia =(IntroductionAdvisor) advisor;  
                if(config.isPreFiltered() || ia.getClassFilter().matches(targetClass)) {  
                    Interceptor[] interceptors= registry.getInterceptors(advisor);  
                    interceptorList.addAll(Arrays.asList(interceptors));  
                }  
           } else {  
                Interceptor[] interceptors =registry.getInterceptors(advisor);  
                interceptorList.addAll(Arrays.asList(interceptors));  
           }  
       }  
       return interceptorList;  
}  
```



```java
//5）这个方法执行完成后，Advised中配置能够应用到连接点或者目标类的Advisor全部被转化成了MethodInterceptor.
//6）接下来货到invoke方法中的proceed方法 ，我们再看下得到的拦截器链是怎么起作用的，也就是proceed方法的执行过程
public Object proceed() throws Throwable {  
       //  We start with an index of -1and increment early.  
       if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size()- 1) {  
           //如果Interceptor执行完了，则执行joinPoint  
           return invokeJoinpoint();  
       }  
   
       Object interceptorOrInterceptionAdvice =  
           this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);  
         
       //如果要动态匹配joinPoint  
       if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher){  
           // Evaluate dynamic method matcher here: static part will already have  
           // been evaluated and found to match.  
           InterceptorAndDynamicMethodMatcher dm =  
                (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;  
           //动态匹配：运行时参数是否满足匹配条件  
           if (dm.methodMatcher.matches(this.method, this.targetClass,this.arguments)) {  
                //执行当前Intercetpor  
                returndm.interceptor.invoke(this);  
           }  
           else {  
                //动态匹配失败时,略过当前Intercetpor,调用下一个Interceptor  
                return proceed();  
           }  
       }  
       else {  
           // It's an interceptor, so we just invoke it: The pointcutwill have  
           // been evaluated statically before this object was constructed.  
           //执行当前Intercetpor  
           return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);  
       }  
}  
```



#### Spring CGLIB动态代理实现

由于CGLIB的动态代理代码量比较长，在这就不贴出来代码了，其实这两个代理的实现方式都差不多，都是创建方法调用链，不同的是jdk的动态代理创建的是

ReflectiveMethodInvocation调用链，而CGLib创建的是CGLib MethodInvocation。

# JDK和CGLIB区别

**JDK动态代理具体实现原理：**

- 通过实现InvocationHandlet接口创建自己的调用处理器；
- 通过为Proxy类指定ClassLoader对象和一组interface来创建动态代理；
- 通过反射机制获取动态代理类的构造函数，其唯一参数类型就是调用处理器接口类型；
- 通过构造函数创建动态代理类实例，构造时调用处理器对象作为参数参入；

JDK动态代理是面向接口的代理模式，如果被代理目标没有接口那么Spring也无能为力，Spring通过Java的反射机制生产被代理接口的新的匿名实现类，重写了其中AOP的增强方法。

**2、CGLib动态代理：**

CGLib是一个强大、高性能的Code生产类库，可以实现运行期动态扩展java类，Spring在运行期间通过 CGLib继承要被动态代理的类，重写父类的方法，实现AOP面向切面编程呢。

**3、两者对比：**

- JDK动态代理是面向接口的。
- CGLib动态代理是通过字节码底层继承要代理类来实现（如果被代理类被final关键字所修饰，那么抱歉会失败）。

**4、使用注意：**

- 如果要被代理的对象是个实现类，那么Spring会使用JDK动态代理来完成操作（Spirng默认采用JDK动态代理实现机制）；
- 如果要被代理的对象不是个实现类那么，Spring会强制使用CGLib来实现动态代理。