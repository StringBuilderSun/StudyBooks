三、AOP原理（1）

## 1、概述

&ensp;&ensp;&ensp;&ensp;Aop是面向接口的，也即是面向方法的，实现是在IOC的基础上，Aop可以拦截指定的方法并且对方法增强，而且无需侵入到业务代码中，使业务与非业务处理逻辑分离，比如Spring的事务，通过事务的注解配置，Spring会自动在业务方法中开启、提交业务，并且在业务处理失败时，执行相应的回滚策略，aop的实现主要包括了两个部分：

- 匹配符合条件的方法(Pointcut)

- 对匹配的方法增强(JDK代理cglib代理)

&ensp;&ensp;&ensp;&ensp;spring针对xml配置和配置自动代理的Advisor有很大的处理差别，在IOC中主要是基于XML配置分析的，在AOP的源码解读中，则主要从自动代理的方式解析，分析完注解的方式，再分析基于xml的方式。

## 2、案例准备

&ensp;&ensp;&ensp;&ensp;下面是spring aop的用法  也是用于源码分析的案例

&ensp;&ensp;&ensp;&ensp;切面类：`TracesRecordAdvisor`

```java
@Aspect
@Component
public class TracesRecordAdvisor {
    
    @Pointcut("execution(* spring.action.expend.aop.services.*.*(..))")
    public void expression() {
    }

    @Before("expression()")
    public void beforePrint()
    {
        System.out.println("进入服务,记录日志开始....");
    }

    @AfterReturning("expression()")
    public void afterPrint()
    {
        System.out.println("退出服务,记录日志退出.....");
    }
}

```

&ensp;&ensp;&ensp;&ensp;xml配置: aop的注解启用只需要在xml中配置这段代码即可，这个是作为入口

```xml
    <aop:aspectj-autoproxy/>
```

&ensp;&ensp;&ensp;&ensp;服务类：`PayServiceImpl` 使用jdk代理  所以要有一个接口

```
@Service
public class PayServiceImpl implements PayService {
    public void payMoneyService() {
        System.out.println("付款服务正在进行...");
    }
}
```

&ensp;&ensp;&ensp;&ensp;测试方法：

```java
   @Test
    public void springAopTestService() {
        ClassPathXmlApplicationContext applicationContext=new ClassPathXmlApplicationContext("spring-aop.xml");
        PayService payService= (PayService) applicationContext.getBean("payServiceImpl");
        payService.payMoneyService();
    }
```

&ensp;&ensp;&ensp;&ensp;执行结果：

```powershell
进入服务,记录日志开始....
付款服务正在进行...
退出服务,记录日志退出.....
```
&ensp;&ensp;&ensp;&ensp;从上面的执行结果看,payMoneyService方法的确是被增强了。 

## 3、BeanFactoryPostProcessor

&ensp;&ensp;&ensp;&ensp;在读spring源码时，我想首先来看下`BeanFactoryPostProcessor`和`BeanPostProcess`,这两个接口都是在spring通过配置文件或者xml获取bean声明生成完`BeanDefinition`后允许我们对生成`BeanDefinition`进行再次包装的入口。

首先看下`BeanFactoryPostProcessor`的定义

```java
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)；
}
```

&ensp;&ensp;&ensp;&ensp;方法`postProcessBeanFactory`的参数为`ConfigurableListableBeanFactory`，前文说过`beanFactory`用来获取bean的，而`ConfigurableListableBeanFactory`继承接口`SingletonBeanRegistry`和`BeanFactroy`，所以可以访问到已经生成过的`BeanDefinitions`集合，如果某个类实现该接口，spring会注册这个类，然后执行这个类的`postProcessBeanFactory`方法，以便我们对`BeanDefinition`进行扩展。

对于`BeanFactoryPostProcessor`只做简单的介绍，只是说明在Spring中，我们可以修改生成后的BeanDefinition，这里住下看下Spring是如何注册`BeanFactoryPostProcessor`并执行`postProcessBeanFactory`的。

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			prepareRefresh();
             //核心方法1
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			prepareBeanFactory(beanFactory);
			try {
				postProcessBeanFactory(beanFactory);
                 //核心方法2 执行BeanFactoryPostProcessor
				invokeBeanFactoryPostProcessors(beanFactory);
				//核心方法 3 注册BeanPostProcessor
				registerBeanPostProcessors(beanFactory);
				// Initialize message source for this context.
				initMessageSource();
				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();
				// Initialize other special beans in specific context subclasses.
				onRefresh();
				// Check for listener beans and register them.
				registerListeners();
				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);
				// Last step: publish corresponding event.
				finishRefresh();
            }
			catch (BeansException ex) {
                 ............
				throw ex;
			}
			finally {
                 ............
				resetCommonCaches();
			}
		}
	}
```

&ensp;&ensp;&ensp;&ensp;核心方法1`obtainFreshBeanFactory`就是前两篇所说的生成`BeanDefinition`的入口，`invokeBeanFactoryPostProcessors`核心方法2就是执行`BeanFactoryPostProcessor`接口的方法。

```java
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
	}
```

&ensp;&ensp;&ensp;&ensp;通过方法getBeanFactoryPostProcessors获取注册BeanFactoryPostProcessor，然后来看看如何添加一个处理器

```java
	@Override
	public void addBeanFactoryPostProcessor(BeanFactoryPostProcessor beanFactoryPostProcessor) {
		this.beanFactoryPostProcessors.add(beanFactoryPostProcessor);
	}
```

&ensp;&ensp;&ensp;&ensp;对于方法`invokeBeanFactoryPostProcessors`不再往下看了，里面的方法大致先对`BeanFactoryPostProcessor`进行排序，排序的标准是是否实现了`PriorityOrdered`，然后根据设置的order大小指定执行顺序，生成一个排序集合和一个普通的集合，最后执行`invokeBeanFactoryPostProcessors`

```java
	private static void invokeBeanFactoryPostProcessors(
			Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

		for (BeanFactoryPostProcessor postProcessor : postProcessors) {
		     //执行到自定义的BeanFactoryPostProcessor
			postProcessor.postProcessBeanFactory(beanFactory);
		}
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法就会循环先前注册的`BeanFactoryPostProcessor`集合，然后执行`postProcessBeanFactory`。

## 4、BeanPostProcess

&ensp;&ensp;&ensp;&ensp;与`BeanFactoryPostProcessor`相比，`BeanPostProcess`就重要得多了，因为Spring的注解、AOP等都是通过这个接口的方法拦截执行的，它贯穿了Bean创建过程的整个生命周期，在IOC阶段，Spring只注册BeanPostProcess，执行则放到了Bean的实例化创建阶段。

   首先看下BeanPostProcessor的接口定义

```java
public interface BeanPostProcessor {
    //在bean创建 属性赋值之后  Aware接口执行之后执行
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    //在init-method afterPropertiesSet 执行之后执行
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}

```

&ensp;&ensp;&ensp;&ensp;在bean的声明周期中，下面的序列是bean创建后要执行的接口和方法顺序：

> 实例化（`autowireConstructor`或者`instantiateBean`)---------------属性初始化(`populateBean`)-------------Aware接
>
> 口（如果bean实现了的话)--------------------------`BeanPostProcess.postProcessBeforeInitialization`------------------
>
> --`PostConstructInitializingBean.afterPropertiesSet`-----`BeanPostProcess.postProcessAfterInitialization`

&ensp;&ensp;&ensp;&ensp;其中通过注解引入依赖的方式就是在`AutowiredAnnotationBeanPostProcessor`这个类中实现的，而接下来要分析的Spring Aop也是从这里开始的，这个类叫`AnnotationAwareAspectJAutoProxyCreator`,

## 5、NameSpaceHanlder

&ensp;&ensp;&ensp;&ensp;在Spring中，任何的技术都是在IOC的基础上进行的，Aop也不例外，程序会首先读取xml配置文件，然后对读取到的标签先查找命名空间，然后找对应的NameSpaceHandler，最终调用parse方法解析标签。

&ensp;&ensp;&ensp;&ensp;aop标签的解析，使用纯注解的方式`aop:aspectj-autoproxy`和使用`aop:config`的配置解析不太一样，具体表现在生成`PointCut`和生成`Before`、`After`、`Around`等切面类时，使用`aop:config`的方式会为这些注解生成一个`BeanDefinition`，而这个`BeanDefinition`的构造函数是由3个`BeanDefinition`组成，表明这个类是合成类，即`synthetic`这个属性为true。然后跟解析普通的bean一样，生成这些实例对象，后面的过程就跟是用纯注解的方式相同了，接下来的分析是基于纯注解分析的，也就是解析从解析`aop:aspectj-autoproxy`这个标签开始。

&ensp;&ensp;&ensp;&ensp;前面的xml文件的标签解析是通过`parseDefaultElement`方法解析默认的`<bean>`标签的，而我们在配置文件里面配置了启动自动代理的方式`<aop:aspectj-autoproxy/>`，当Spring读取到这个标签，则会走`parseCustomElement(root)`这个方法了，这个方法的源码不再解析，主要完成的功能如下:

- 获取element的nameSpaceUri,根据nameSpaceUri找到NameSpaceHanlder

- 调用NameSpaceHanlder的parse方法解析element


&ensp;&ensp;&ensp;&ensp;下面是NameSpaceHanlder接口的定义

```java
public interface NamespaceHandler {
	void init();
	BeanDefinition parse(Element element, ParserContext parserContext);
	BeanDefinitionHolder decorate(Node source, BeanDefinitionHolder definition, ParserContext       parserContext);
}

```

&ensp;&ensp;&ensp;&ensp;这里面的init方法是我们初始化操作的，这里可以完成对指定的标签设置解析器，然后再parse方法里面找到指定标签的解析器，然后调用该解析器的parse方法解析标签，后面会重点看这两个方法。

&ensp;&ensp;&ensp;&ensp;再来看下Spring如何加载NameSpaceHanlder的，Spring首先会取查找项目空间下目录META-INF/的所有spring.handlers文件，这个文件是在Spring依赖的jar下面，在核心jar包都会由这个文件，aop的jar包路径下文件内容为：spring.handlers

  ```java
  http\://www.springframework.org/schema/aop=org.springframework.aop.config.AopNamespaceHandler
  ```

&ensp;&ensp;&ensp;&ensp;发现这里面存储的是一个key，value,key是aop的nameSpaceUri，value是AopNamespaceHandler，从这个类名上就能发现该类实现了NamespaceHandler，肯定也就实现了init和parse方法，所以解析`<aop:aspectj-autoproxy/>`的任务就由AopNamespaceHandler的parse完成。

查看AopNamespaceHandler的init方法

```java
@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new                       AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());
		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}
```

上面的代码就很清晰了，`<aop:config>`标签由`ConfigBeanDefinitionParse`r处理，`<aop:aspectj-autoproxy/>`则由`AspectJAutoProxyBeanDefinitionParser`这个类处理，这两种处理其实对应了自动代理和通过xml配置的处理方式，然后会调用`AspectJAutoProxyBeanDefinitionParser`的`parse`方法

```java
	@Override
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
		extendBeanDefinition(element, parserContext);
		return null;
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法其实就是为了注册一个`AnnotationAwareAspectJAutoProxyCreator`类，然后AOP的所有处理逻辑都会交给这个类处理,由于这个类的实现了`BeanPostProcessor`,所以这个类的入口就是BeanPostProcessor接口的两个方法：

- postProcessBeforeInitialization
- postProcessAfterInitialization

## 6、Spring Aop 源码解读前奏

&ensp;&ensp;&ensp;&ensp;上面分析了，当spring读取xml文件遇到`<aop:aspectj-autoproxy/>`会找到`AopNamespaceHandler`这个处理类，然后这个类又将这个标签委托给了`AspectJAutoProxyBeanDefinitionParser`类，最终调用这个类得parse方法，`parse`方法未做分析，其实这个方法的目的很简单，就是注册`AnnotationAwareAspectJAutoProxyCreator`这个类，这个类实现了`BeanPostProcessor`和`InstantiationAwareBeanPostProcessor`接口，最终在实例化bean对象也就是执行`BeanFactory.getBean(beanName)`的过程中，会调用这两个接口的方法(执行顺序如下)：

>  InstantiationAwareBeanPostProcessor先执行：
>
> `postProcessBeforeInstantiation(Class<?> beanClass, String beanName) `
>
> ` postProcessAfterInstantiation(Object bean, String beanName)`
>
> BeanPostProcessor再执行：
>
> `postProcessBeforeInitialization(Object bean, String beanName)`
>
> `Object postProcessAfterInitialization(Object bean, String beanName)`

 &ensp;&ensp;&ensp;&ensp;AOP的实现基本上是在这两个方法中进行的，所以就从这里来看Spring是如何实现AOP的，Spring的AOP代理目前支持方法的增强，看源码目前好像也支持了属性的增强了。

 &ensp;&ensp;&ensp;&ensp;读取源码前首先来分析一下方法增强的原理，有助于我们读取源码时紧紧抓住主线。首先第一个问题，如果我们想对一个类的方法进行增强，我们应该怎么做呢？

&ensp;&ensp;&ensp;&ensp;这种业务需求可以通过代理实现，在方法执行前，拦截这个方法，并且加入要执行增强的逻辑，最后再执行目标方法。下面是Spring用的两种代理方式：

>JDK代理：我们可以通Proxy类获取一个目标类的代理对象，但JDK代理要求被代理的类必须实现接口，所以是基于接口的代理。

>cglib代理：如果目标类没有接口，使用cglib代理，是由asm封装的，直接操作类得字节码，效率也很高。

&ensp;&ensp;&ensp;&ensp;由于在生产业务中，我们不可能对所有的类都执行增强，所以还需要一个选择器，将符合条件的bean进行增强，Spring使用了`PointCut`接口，通过该接口的`getMethodMatcher`方法获取一个方法匹配器，然后通过`matches`方法匹配到目标类对象的目标方法执行增强操作。`mathcer`匹配规则就是通过Spring 配置的expression表达式了。

&ensp;&ensp;&ensp;&ensp;所以在分析源码的时，要围绕这两方面进行：

- 匹配切点方法（构建切入点表达式类和切面类）

- 创建代理对象 

&ensp;&ensp;&ensp;&ensp;这两方面在Spring的实现里非常复杂，尤其是第一步匹配切点方法过程，这个过程中，Spring会将`@Aspect`注解类的`@Before`，`@After`，`@Around`、`@Pointcut`等注解都封装成待执行的切面方法类，然后通过方法匹配器匹配到的要增强的方法前后执行切面方法类，达到方法增强的目的。

&ensp;&ensp;&ensp;&ensp;第二阶段，创建代理对象默认是通过JDK代理实现配置，`<aop:aspectj-autoproxy proxy-target-class="true">`这样配置可以指定使用cglib代理。

## 7、注解切面代理类

&ensp;&ensp;&ensp;&ensp;上面分析了真正实现AOP功能的是`AnnotationAwareAspectJAutoProxyCreator`,由于这个类实现了`BeanPostProcessor`和`InstantiationAwareBeanPostProcessor`，所以在创建一个bean的时候，会进入到这两个接口的方法，这两个接口包含了四个方法，方法执行顺序上面已经分析过了，来看看这个类的类图：

![](C:\Users\ljp\Desktop\mds\Spring\AnnotationAwareAspectJAutoProxyCreator.png)

&ensp;&ensp;&ensp;&ensp;类图上比较重要的接口就是右上角实现的两个接口，在bean创建的生命周期过程中，会校验当前容器中是否注册了实现了这两个接口的类，如果有则调用接口的方法，前面的分析中在解析`<aop:aspectj-autoproxy/>`时，将这个类注册到了容器中，而且上面也罗列了这两个接口中四个方法的调用顺序，在这个类中完成主要功能的2个方法及其执行顺序:

>InstantiationAwareBeanPostProcessor先执行：
>
>`postProcessBeforeInstantiation(Class<?> beanClass, String beanName) `
>
>BeanPostProcessor再执行：
>
>`Object postProcessAfterInitialization(Object bean, String beanName)`

&ensp;&ensp;&ensp;&ensp;`postProcessBeforeInstantiation`方法主要是找出注解了Advice的类，并将Advice的类使用了`@Before`，`@After`，`@Around`、`@Pointcut`,`@AfterThrowing`等注解的方法封装成一个一个类放入到缓存中供匹配到的类生成代理用。`postProcessAfterInitialization`主要是匹配符合条件的目标类对象，然后生成代理的过程,接下来就按顺序分析这两个方法完成的功能。 

## 8、处理Aspect注解类

```java
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        //构建一个缓存key
		Object cacheKey = getCacheKey(beanClass, beanName);
		if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
             //如果当前beanClass的缓存key 存在于Class为Advise的缓存中，表示当前的beanClass是Adivse类
             //而且不需要生成代理。
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
             //核心校验：1 当前类是否是AOP的基础类 2、当前类是否应该跳过不生成代理
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}
        //这部分主要是用于实现了TargetSource接口的bean，然后从getTarget中获取对象 创建代理
		if (beanName != null) {
			TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
			if (targetSource != null) {
				this.targetSourcedBeans.add(beanName);
				Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
				Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
				this.proxyTypes.put(cacheKey, proxy.getClass());
				return proxy;
			}
		}
		return null;
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法主要是先为`beanClass`生成一个缓存的key，这个`beanClass`如果是`FactoryBean`，则按照工厂类的命名规则命名，否则用`beanName`命名，然后用刚才生成的key判断`beanClass`是否已经存在于`Advice`的缓存集合中，如果已经存在则代表该类是切面类而且已经被处理过了，后续处理不会为该类生成代理，如果没有没处理过，则会调用下面的方法校验该类是否是AOP的基础类 ，总之这个方法作用就是将AOP相关操作的切面类和基础类放入到缓存中，当为bean生成代理的时候，忽略`advice`缓存中的AOP切面类和基础类，下面是具体校验过程：

`AnnotationAwareAspectJAutoProxyCreator`重写了该方法

```java
@Override
protected boolean isInfrastructureClass(Class<?> beanClass) {
//调用父类的isInfrastructureClass判断是否是aop基础类
//校验当前类是否使用@Aspect注解
return (super.isInfrastructureClass(beanClass) || this.aspectJAdvisorFactory.isAspect(beanClass));
}
```
父类的isInfrastructureClass方法
```java
	protected boolean isInfrastructureClass(Class<?> beanClass) {
		boolean retVal = Advice.class.isAssignableFrom(beanClass) ||
				Advisor.class.isAssignableFrom(beanClass) ||
				AopInfrastructureBean.class.isAssignableFrom(beanClass);
		if (retVal && logger.isTraceEnabled()) {
			logger.trace("Did not attempt to auto-proxy infrastructure class [" + beanClass.getName() + "]");
		}
		return retVal;
	}
```

&ensp;&ensp;&ensp;&ensp;里面`isAssignableFrom`表示当前类是否允许被设置为`beanClass`类对象，可以以此判断`beanClass`是否是Advice类，所以这个方法的校验目的就是判断当前正在创建目标类是否是AOP的基础类，即该类是否是`Advice`,`Advisor`或者实现了`AopInfrastructureBean`接口。该方法调用父类的`isInfrastructureClass`判断是否是aop基础类，然后再校验当前类是否使用`@Aspect`注解，目的只有一个，如果是`Advice`切面相关的类不做任何处理，直接放入`advice`缓存即可。

然后再来看`shouldSkip(beanClass, beanName)`：

```java
	@Override
	protected boolean shouldSkip(Class<?> beanClass, String beanName) {
		//查找当前已经生成的所有Advisor切面类  不展开分析
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		for (Advisor advisor : candidateAdvisors) {
			if (advisor instanceof AspectJPointcutAdvisor) {
				if (((AbstractAspectJAdvice) advisor.getAdvice()).getAspectName().equals(beanName)) {
					return true;
				}
			}
		}
		return super.shouldSkip(beanClass, beanName);
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法主要是校验当前正在创建bean的beanName是否属于已经创建好的切面类缓存中，如果是则加入到`advices`缓存中，不再处理。其中`findCandidateAdvisors()`会查找当前容器中生成的所有实现了`Advisor`的类，Spring会将`@Before`，`@After`，`@Around`等生成一个继承了Advisor类对象存储到缓存中供后续使用,这一部分时Spring AOP前半段的核心内容，后续都会围绕着如何将切面类的注解生成Adisor类探索。

`AnnotationAwareAspectJAutoProxyCreator`重写了`findCandidateAdvisors`方法，所以会执行到该方法：

```java
@Override
protected List<Advisor> findCandidateAdvisors() {
   //通过父类的方法查找所有容器中的Advisor类，也就是基于xml配置的<aop:before/>生成的
   List<Advisor> advisors = super.findCandidateAdvisors();
   //查找通过注解的方式生成Advisor类
   advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
   return advisors;
}
```

&ensp;&ensp;&ensp;&ensp;这个方法会首先调用父类的`findCandidateAdvisors`方法用于获取通过xml文件配置生成的`Advisor`，也就是通过`  <aop:before>`,`<aop:after>`等生成的，然后调用通过注解方式即`@Before`，`@After`，`@Around`、`@Pointcut`,`@AfterThrowing`生成的advisor，可以说，这两个方法分别处理了基于xml配置文件的方式和基于注解的配置方式，因为所有的分析都是基于AnnotationAwareAspectJAutoProxyCreator这个类进行的，所以在这个地方会先获取配置文件的，再生成基于注解类的Advisor，这样就将基于xml配置的和基于注解的配置都会解析到。

看下`this.aspectJAdvisorsBuilder.buildAspectJAdvisors()`

```java
	public List<Advisor> buildAspectJAdvisors() {
		List<String> aspectNames = null;
		synchronized (this) {
			aspectNames = this.aspectBeanNames;
			if (aspectNames == null) {
				List<Advisor> advisors = new LinkedList<Advisor>();
				aspectNames = new LinkedList<String>();
                //从beanDefinitions中获取所有的beanName
				String[] beanNames =
BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Object.class, true, false);
				for (String beanName : beanNames) {
                      //如果beanName不符合配置的 <aop:include name="***"/>
                      //忽略这个bean上所有的切面方法
					if (!isEligibleBean(beanName)) {
						continue;
					}
					Class<?> beanType = this.beanFactory.getType(beanName);
					if (beanType == null) {
						continue;
					}
                      //如果当前beanType是一个切面类 则将该切面类相关信息封装起来
					if (this.advisorFactory.isAspect(beanType)) {
						aspectNames.add(beanName);
						AspectMetadata amd = new AspectMetadata(beanType, beanName);
						if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {                   
//将切面信息放入到分装到MetadataAwareAspectInstanceFactory 生成一个AspectMetadata
MetadataAwareAspectInstanceFactory factory =new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
//获取容器中所有Advisor类 需要进入这个方法详细分析
List<Advisor> classAdvisors=this.advisorFactory.getAdvisors(factory);
							if (this.beanFactory.isSingleton(beanName)) {
                                   //单例加入缓存
								this.advisorsCache.put(beanName, classAdvisors);
							}
							else {
                                   //非单例  将工厂加入缓存
								this.aspectFactoryCache.put(beanName, factory);
                            }
							advisors.addAll(classAdvisors);
						}
						else {
							// 非单例  将生成Advisor的工厂类加入到缓存
							if (this.beanFactory.isSingleton(beanName)) {
								throw new IllegalArgumentException("Bean with name '" + beanName +
										"' is a singleton, but aspect instantiation model is not singleton");
							}
							MetadataAwareAspectInstanceFactory factory =
									new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
							this.aspectFactoryCache.put(beanName, factory);
							advisors.addAll(this.advisorFactory.getAdvisors(factory));
						}
					}
				}
				this.aspectBeanNames = aspectNames;
				return advisors;
			}
		}
        ..............
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法主要的任务其实就是获取类得类型为`Aspect`的切面类，然后获取切面类方法的所有注解并将注解转换成`Advisor`类返回，主要步骤为：

- 获取容器中所有的`BeanDefinition的beanName`

- 根据`beanName`，或者`beanClass`，匹配符合规则的Aspect切面类，通过`<aop:include>`配置的规则

- 获取`Aspect`切面类的所有切面方法封装成`Advisor`对象返回。

- 将获取到的所有`Advisor`放入到缓存中。

&ensp;&ensp;&ensp;&ensp;这个方法代码虽然很多，但是核心的是`this.advisorFactory.getAdvisors(factory)`，即第三个步骤，这个方法将会获取到切面类的所有切面方法，并封装成`Advisor`，`getAdvisors`是一个接口，`ReflectiveAspectJAdvisorFactory`实现了这个接口，下面代码是其实现逻辑：

```java
	@Override
	public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
         //获取切面类Class
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
        //获取切面类的beanName
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
		validate(aspectClass);
	    //进一步对AspectMetadata封装 里面包含了切面类的信息
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);
		List<Advisor> advisors = new LinkedList<Advisor>();
         //获取切面类中没有使用Pointcut注解的方法
		for (Method method : getAdvisorMethods(aspectClass)) {
             //检查该方法是否是切面方法， 如果是成Advisor类返回
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}
		//如果没有切面方法 设置一个空的
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}
		//处理属性字段 Spring支持到了属性的增强
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}
		return advisors;
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法首先已经将切面类信息封装到`AspectMetadata`的类再次封装到`MetadataAwareAspectInstanceFactory`，然后获取切面类的所有没有使用`Pointcut`注解的方法，调用`getAdvisor`获取这个方法使用的切面注解，生成对应的`Advisor`类。 至于`PointCut`的处理则是再后面的`getAdvisor`中处理的。

## 9、获取切面类的Advisor

&ensp;&ensp;&ensp;&ensp;获取`Advisor`类的方法为`getAdvisor`，首先来看下这个方法的参数：

```java
//切面类的切面方法  这里可能就是 beforePrint()
Method  candidateAdviceMethod 
//获取AspectMetadata的实例工厂(可以获取切面的类所有信息)
MetadataAwareAspectInstanceFactory aspectInstanceFactory
//切面的排序
int declarationOrderInAspect
//切面类的beanName 这里是tracesRecordAdvisor
 String aspectName
```

&ensp;&ensp;&ensp;&ensp;上面的参数中可以获取到切面类和切面方法，这样就可以获得一个Advisor对象，然后还需要一个切入点表达式`PointCut`用来匹配符合条件的方法，拦截到目标方法后，就可以执行`Adivsor`增强方法了。 来看看创建`Advisor`的过程，这里假设`Method`是`TracesRecordAdvisor`类的`beforePrint`方法，也就是我们测试案例中创建使用了`@Before`注解的切面方法：

```java
@Override
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
      int declarationOrderInAspect, String aspectName) {

   validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
   //获取pointCut,这里实际上获得的是 expression()这个方法对应了pointCut的内容
   AspectJExpressionPointcut expressionPointcut = getPointcut(
         candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
   if (expressionPointcut == null) {
      return null;
   }
    //创建advisor
   return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
         this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```
&ensp;&ensp;&ensp;&ensp;看看`getPointCut`方法如何获取到`exression`过程需要嵌套很多步骤，这里不展开了，简单看下如何将查找到的值设置到表达式中的：

```java
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
   //
   AspectJAnnotation<?> aspectJAnnotation =
         AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
   if (aspectJAnnotation == null) {
      return null;
   }

   AspectJExpressionPointcut ajexp =
         new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
   //将上面生成的AspectJAnnotation 解析出的expression方法放入到表达式中
   //
   ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
   return ajexp;
}
```
&ensp;&ensp;&ensp;&ensp;这里需要关注下上面的`findAspectJAnnotationOnMethod`方法：

```java
protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
    //看到了我们熟悉的切面方法注解，这里的beforePrint使用@Before注解
   Class<?>[] classesToLookFor = new Class<?>[] {
         Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.class};
   for (Class<?> c : classesToLookFor) {
      AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) c);
      if (foundAnnotation != null) {
         return foundAnnotation;
      }
   }
   return null;
}
```

&ensp;&ensp;&ensp;&ensp;这个方法就是查找切面方法是否使用了`Before`, `Around`, `After`，`AfterReturning`, `AfterThrowing`,` Pointcut`注解，如果使用了，则返回一个`AspectJAnnotation`对象，里面有一个`annotation`的泛型对象，这个泛型对象就是被设置为这些注解的值，而且还会获得这些注解里面配置的`pointcut`表达式内容，如果是引用的表达式方法，则将方法参数设置到`pointcutExpression`这个属性中。

&ensp;&ensp;&ensp;&ensp;解析完切面方法的注解后现在再回过头来看看如何创建一个`advisor`实例：

```java
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
      Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
      MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
   this.declaredPointcut = declaredPointcut;
   this.aspectJAdviceMethod = aspectJAdviceMethod;
   this.aspectJAdvisorFactory = aspectJAdvisorFactory;
   this.aspectInstanceFactory = aspectInstanceFactory;
   this.declarationOrder = declarationOrder;
   this.aspectName = aspectName;
    //切面类是否是懒加载
   if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
      // Static part of the pointcut is a lazy type.
      Pointcut preInstantiationPointcut = Pointcuts.union(
            aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

      // Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
      // If it's not a dynamic pointcut, it may be optimized out
      // by the Spring AOP infrastructure after the first evaluation.
      this.pointcut = new PerTargetInstantiationModelPointcut(
            this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
      this.lazy = true;
   }
   else {
      this.pointcut = this.declaredPointcut;
      this.lazy = false;
       //最终会执行到这里获取一个advice
      this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
   }
}
```
## 10、为切面方法创建Advice

&ensp;&ensp;&ensp;&ensp;上面方法的最后一句`instantiateAdvice(this.declaredPointcut)`会创建一个`advice`，具体是调用`getAdvice`方法获取：

```java
@Override
	public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
         //获取切面类对象，这里是TracesRecordAdvisor
		Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		validate(candidateAspectClass);
          //核心点1：获取切面注解，这里得方法是 beforePrint 使用了@Before注解
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}
            ....................
		AbstractAspectJAdvice springAdvice;
        //核心点2：根据注解转换后的 将注解生成不同的Advice类。
		switch (aspectJAnnotation.getAnnotationType()) {
			case AtBefore:
				springAdvice = new AspectJMethodBeforeAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfter:
				springAdvice = new AspectJAfterAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfterReturning:
				springAdvice = new AspectJAfterReturningAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
					springAdvice.setReturningName(afterReturningAnnotation.returning());
				}
				break;
			case AtAfterThrowing:
				springAdvice = new AspectJAfterThrowingAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
				}
				break;
			case AtAround:
				springAdvice = new AspectJAroundAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtPointcut:
                //这里对PointCut不做处理
				if (logger.isDebugEnabled()) {
					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
				}
				return null;
			default:
				throw new UnsupportedOperationException(
						"Unsupported advice type on method: " + candidateAdviceMethod);
		}

		// 将切面类信息配置到SpringAdvice中
		springAdvice.setAspectName(aspectName);
		springAdvice.setDeclarationOrder(declarationOrder);
		String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
		if (argNames != null) {
			springAdvice.setArgumentNamesFromStringArray(argNames);
		}
		springAdvice.calculateArgumentBindings();
		return springAdvice;
	}
```

&ensp;&ensp;&ensp;&ensp;首先来看看核心点1，上面其实已经看过了， 但是上面的方法作用仅仅是为了获取注解上的`exression`表达式的，这里再调用一遍就是为注解生成`Advice`类的，目的就是获取切面注解与`AspectJAnnotation`的映射类。

```java
protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
    //看到了我们熟悉的切面方法注解，这里的beforePrint使用@Before注解
   Class<?>[] classesToLookFor = new Class<?>[] {
         Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.class};
   for (Class<?> c : classesToLookFor) {
      AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) c);
      if (foundAnnotation != null) {
         return foundAnnotation;
      }
   }
   return null;
}
```

&ensp;&ensp;&ensp;&ensp;这个方法就是查找切面方法是否实现了`Before`, `Around`, `After`，`AfterReturning`, `AfterThrowing`,` Pointcut`注解，如果实现了，则返回一个`AspectJAnnotation`对象，里面有一个`annotation`的泛型对象，这个泛型对象就是被设置为这些注解的值。最终这些对象会被转换成下面的对象存入`AspectJAnnotation`中：

```java
static {
   //会将注解转换成后面的AspectJAnnotationType枚举的类。 
   annotationTypes.put(Pointcut.class,AspectJAnnotationType.AtPointcut);
   annotationTypes.put(After.class,AspectJAnnotationType.AtAfter);
   annotationTypes.put(AfterReturning.class,AspectJAnnotationType.AtAfterReturning);
   annotationTypes.put(AfterThrowing.class,AspectJAnnotationType.AtAfterThrowing);
   annotationTypes.put(Around.class,AspectJAnnotationType.AtAround);
   annotationTypes.put(Before.class,AspectJAnnotationType.AtBefore);
}
```

&ensp;&ensp;&ensp;&ensp;通过核心点1，Spring已经将注解`@Before`对应转换为`AtBefore`，`@After`转换成`AtAfter`，以此类推，都会一一映射到了核心点2的switch的条件类了，在核心点2中，会为对应的切面注解类生成`Advice`类。 所有的注解切面类具体实现都是由`AbstractAspectJAdvice`这个抽象类实现的，这个类的构造函数有三个参数：

```java
//切面方法 这里可能是beforePrint
Method aspectJAroundAdviceMethod
//切入点表达式匹配器 这里指封装了exression的匹配器
AspectJExpressionPointcut pointcut
//切面类  这里指TracesRecordAdvisor
AspectInstanceFactory aif
```

&ensp;&ensp;&ensp;&ensp;下面是Spring为对应注解生成对应的Advice类

|      注解类      |       Advice 顾问方法       |
| :--------------: | :-------------------------: |
|     AtBefore     |  AspectJMethodBeforeAdvice  |
|     AtAfter      |     AspectJAfterAdvice      |
| AtAfterReturning | AspectJAfterReturningAdvice |
| AtAfterThrowing  | AspectJAfterThrowingAdvice  |
|     AtAround     |     AspectJAroundAdvice     |

&ensp;&ensp;&ensp;&ensp;各个注解会在不同的实际执行自身增强方法，这个部分只是生成`Advice`类，然会放入到缓存中，等真正生成代理时就会调用这些方法。这个在创建代理的时候需要具体拆开说，至此，Spring将使用了`@Aspect`注解的切面类的切面方法，都转换成了对应的`Adivsor`类，这个类包含了切面方法，封装后的切点匹配器`PointCut`以及生成切面类的实例对象，通过这个类就可以匹配到符合条件的目标类的目标方法，然后执行增强操作了。 

&ensp;&ensp;&ensp;&ensp;由切面注解生成的`Advice`类，最终会放入到一个缓存中，当生成目标bean的时候，会将所有所以能够匹配到目标bean的advice放入到集合中，由一个实现了`MethodInvocation`的类统一管理调用过程，这个类后面会详细说到，这里简单分析下`AspectJAfterAdvice`的invoke方法，看看它的调用过程

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
             //调用是实现了MethodInvocation方法的类 这个其实是个链式调用
			return mi.proceed();
		}
		finally {
            //最终执行后置增强方法
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}
```

&ensp;&ensp;&ensp;&ensp;上面的`invoke`方法需要一个`MethodInvocation`的参数，上面的`Advice`类除了`AspectJMethodBeforeAdvice`之外，都实现了这个接口，所以可以实现链式调用，这个逻辑会在创建代理的具体讲解，这里只是简单分析下，这些`advice`的`invoke`方法规定了切面方法于要增强方法的执行时机。

## 11、AOP代理初窥

&ensp;&ensp;&ensp;&ensp;上面一部分操作主要是处理使用了`@Aspect`注解的切面类，然后将切面类的所有切面方法根据使用的注解生成对应的`Advisor`的过程，这个`Advisor`包含了切面方法，切入点匹配器和切面类，也就是准好了要增强的逻辑，接下来就是要将这些逻辑注入到合适的位置进行增强，这部分的操作就是由老生常谈的代理实现的了。 

```Java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
       //如果要创建的类不是提前暴露的代理 则进入下面的方法
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```
&ensp;&ensp;&ensp;&ensp;创建代理前，需要先校验bean是否需要创建代理

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    //如果bean是通过TargetSource接口获取 则直接返回
   if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
    //如果bean是切面类 直接返回
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
    //如果bean是Aspect 而且允许跳过创建代理， 加入advise缓存 返回
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }
   //如果前面生成的advisor缓存中存在能够匹配到目标类方法的Advisor 则创建代理
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
       //创建代理
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```

&ensp;&ensp;&ensp;&ensp;方法很简单，主要的关注点在`getAdvicesAndAdvisorsForBean`和`createProxy`上，第一个是获取能够匹配目标类方法的`Advisor`集合，如果这个集合不为空，则代表该类需要被增强，需要生成代理，如果匹配不到，则表示该类并不需要被增强，无需创建代理。至于`createProxy`就很明显了，就是创建代理，这个方法里面决定了使用jdk代理还是cglib代理，并且用到了前面生成的Advisor实现增强功能。 这部分内容会放到下一篇文章中专门分析。 

## 12、简单总结

&ensp;&ensp;&ensp;&ensp;简单总结一下，Spring AOP在第一阶段完成的主要任务：

&ensp;&ensp;&ensp;&ensp;读取配置文件阶段：

- 读取xml文件遇到  ` <aop:aspectj-autoproxy/>`标签时，找到命名空间处理器`AopNamespaceHandler`,然后找到处理该标签的类`AspectJAutoProxyBeanDefinitionParser`

- 通过`AspectJAutoProxyBeanDefinitionParser`的`parse`方法，将`AspectJAwareAdvisorAutoProxyCreator`注册到容器的声明周期中。

&ensp;&ensp;&ensp;&ensp;创建bean阶段：

- 执行`AspectJAwareAdvisorAutoProxyCreator`的`postProcessBeforeInstantiation`校验目标类是否是`Aspect`类和AOP基础类以及是否需要跳过不需要执行代理的类

- 获取`beanDefinitions`中所有使用了Aspect注解的类，然后将切面方法根据使用的注解生成Advisor类放入到缓存（关键）

- 调用`AspectJAwareAdvisorAutoProxyCreator`的`postProcessAfterInitialization`的方法，对需要增强的类创建代理。

&ensp;&ensp;&ensp;&ensp;这个就是Spring AOP在这个阶段所完成的工作，下一部分将专门针对Spring如何实现jdk和cglib代理分析。