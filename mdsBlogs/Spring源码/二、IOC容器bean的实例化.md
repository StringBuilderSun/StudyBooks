# 二、IOC容器bean的实例化

## 1、概述

&ensp;&ensp;&ensp;&ensp;上一节分析了Spring如何读取xml配置文件并最终将配置的POJO类生成一个个`BeanDefinition`注册到IOC容器的过程，主要是针对直接配置在xml中的<bean name="**" class="***"/>标签来分析的，应该来说生成BeanDefinition指数读取配置放入到指定属性中，并不是太难理解。
&ensp;&ensp;&ensp;&ensp;IOC的第二步是通过`getBean()`获取一个bean实例，相对而言，创建一个bean比生成一个`BeanDefinition`热闹多了，bean的singleton和prototype模式，bean的属性注入，bean的循环依赖，bean方法拦截等等都是创建一个bean要解决的问题，而这一些对于使用者来说无需关注，但如果能了解Spring的实现原理，遇到问题才不会手忙脚乱。

## 2、BeanFactory

&ensp;&ensp;&ensp;&ensp; `BeanFactory`是Spring提供给我们用于获取bean实例的接口，这个接口很简单，没有多余的方法，只有一个非常核心的getBean()，通过该方法就可以获取一个bean的实例。在Spring中该接口是由`DefaultListableBeanFactory`实现，真正的业务处理在`AbstractBeanFactory`类里面，再来回忆下这张类图:

![](C:\Users\ljp\Desktop\mds\Spring\DefaultListableBeanFactory.png)

`getBean`的入口在`DefaultListableBeanFactory`，准确来说是在`AbstractBeanFactory`这个类里面，因为getBean()的很多实现都是在`AbstractBeanFactory`这个抽象类里面。

## 3、案例准备

&ensp;&ensp;&ensp;&ensp; 再来看一下测试案例：     

```java
    @Test
    public void testSpringLoad() {
        ClassPathXmlApplicationContext application = new ClassPathXmlApplicationContext("spring/spring-context.xml");
        Person person= (Person) application.getBean("person");
        BeanService beanService= (BeanService) application.getBean("beanService");
        System.out.println(beanService);
        System.out.println(person);
    }
```

在这个测试案例里，需要通过application获取到一个person和beanService的实例，这个实例在xml的配置信息如下:

```java
    <!--setter注入-->
    <bean id="beanService" class="com.yms.manager.model.BeanService">
        <property name="mapper" ref="userDao"/>
        <property name="name" value="lijinpeng"/>
        <property name="sex" value="false"/>
    </bean>
    <!--构造器注入-->
    <bean id="person" class="com.yms.manager.model.Person">
        <constructor-arg name="age" value="26"/>
        <constructor-arg name="name" value="dangwendi"/>
        <constructor-arg name="userDao" ref="userDao"/>
        <constructor-arg name="sex" value="true"/>
    </bean>
    <bean id="userDao" class="com.yms.manager.model.UserDao"/>
```

通过这些配置，需要得到预期的结果:

- 获取beanService实例，并通过property将属性注入类实例
- 获取person实例，并通过constructor-arg将属性注入类实例
  

 &ensp;&ensp;&ensp;&ensp; 这两种方式实际上对应了setter注入和构造器注入，由于getBean()里面的实现非常复杂，针对注解和aop使用了几个`BeanPostProcessor`在bean的不同生命周期执行注入，里面用的类很多，如果把所有的情况都分析下来，估计很快就懵了，所以还是通过这个简单的案例来分析一下Spring是如何实例化bean的，在分析原理时相对而言容易理解一点。

## 4、getBean思路分析

 &ensp;&ensp;&ensp;&ensp;BeanFactory里面通过getBean获取bean实例，先来看一下getBean的几个重载:

- `getBean(String name)`

- `getBean(Class<T> requiredType)`

- `getBean(String name, Class<T> requiredType)` 

- `getBean(String name, Object... args)`

&ensp;&ensp;&ensp;&ensp;这四个重载方法都用于获取bean实例，通过参数名就可以看出，name就是bean实例化时候的beanId，这个id的值如果通过xml配置就是bean的name属性，如果是注解就是自动生成的，默认是类名的首字母小写。requiredType是需要获取指定Class的bean，这里重点了解一下最后一个重载，这个方法第二个参数是一个动态数组，这个动态数组主要用于指定创建bean实例时候要使用的构造函数的参数列表或者创建bean的工厂方法返回bean的方法参数列表，这也就意味着，我们可以在使用的bean的时候可以通过指定不通的构造参数获取bean的不同对象，常用于工厂模式。

&ensp;&ensp;&ensp;&ensp; 在看源码以前，首先来想一下如果自己简单实现，应该如何获取一个bean，前面说过IOC实际上就是一个线程安全的hashMap，准确来说是 `ConcurrentHashMap<String, Object>`,知道这个那就简单了，所以大概应该是这个步骤：

- 通过beanName去第一步生成的BeanDefinitions查找`BeanDefinition`

- 从`BeanDefinition`获取className，匹配一个合适的构造函数，通过反射获取一个实例对象

- 从`BeanDefinition`获取配置的属性值，通过反射注入到实例对象中

- 返回bean实例

&ensp;&ensp;&ensp;&ensp;上面的分析步骤是针对getBean(String name) 这个重载方法的，如果是`getBean(Class<T> requiredType)`则在第二步需要变动一下:

&ensp;&ensp;&ensp;&ensp;`BeanDefinition`提供一个方法`resolveBeanClass`将配置className解析成beanClass对象，然后从BeanDefinitions查找beanClass为requiredType的对象返回实例。然后进行3、4步骤即可。

&ensp;&ensp;&ensp;&ensp;Spring的实现方式实际上复杂得多,简单来说Spring是这么处理得:

- 先从缓存中查找是否已经存在解析过的名为name，如果有，直接返回

- 如果bean是`FactoryBean`或者是bean工厂类，调用getObject()或者工厂方法获取bean实例

- 匹配合适的构造函数，通过反射创建一个bean的实例

- 通过`ObjectFactory`提前暴露bean，解决循环依赖的问题

- 调用`populateBean`方法完成bean的属性注入

- 将实例化完成的bean放入到IOC容器中

&ensp;&ensp;&ensp;&ensp;spring中bean的获取，针对注解注入和AOP的入口都是在实现了`BeanPostProcessor`的类里实现的，这些类共同组成了bean实例化过程中的一部分生命周期，通过上面对getBean的简要分析，接下来就去看看Spring的源码部分吧。

&ensp;&ensp;&ensp;&ensp;上面说过，`AbstractBeanFactory`是创建bean实例的程序入口，具体到方法是`doGetBean`，由于这个方法比较长，这里只分析比较重要的逻辑，下面的方法比源码有些精简,不过核心业务都是有的：

```java
protected <T> T doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly){
//解析bean的真正name，如果bean是工厂类，name前缀会加&，需要去掉
final String beanName = transformedBeanName(name);
//核心1：从单例容器中获取缓存的bean
 Object sharedInstance = getSingleton(beanName);
 //如果从单例容器中到缓存的bean而且构造函数参数列表为空
if (sharedInstance != null && args == null) 
{   
    //核心2：直接获取实例
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}else{
//验证当前bean实例是否正在创建中，如果是 直接抛出异常
if (isPrototypeCurrentlyInCreation(beanName)) {
throw new BeanCurrentlyInCreationException(beanName);
  }
    //中间省略了通过工厂方式创建bean的代码
    .............
if (!typeCheckOnly)
{
    //核心3：将当前bean实例放入alreadyCreated集合里，标识这个bean准备创建了
    markBeanAsCreated(beanName);
}
    try{
        //从BeanDefinitions中获取该实例的BeanDefinition
        final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        //检查要创建的实例bean的修饰符，是否允许被实例化
	    checkMergedBeanDefinition(mbd, beanName, args);
        //省略了实例化bean的所有依赖bean的过程,创建过程是一样的
         .............
             // Create bean instance.
		if (mbd.isSingleton()) {
         //核心4：创建单例bean
		sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
         //为了解决循环依赖，spring将使用ObjectFactory表示bean，提前暴露bean
         //当真正属性注入的时候，调用ObjectFactory的getObject()方法获取bean实例
		@Override
		public Object getObject() throws BeansException {
		try {
         //核心5：真正创建bean的方法，重点要看的方法   
		return createBean(beanName, mbd, args);
		}catch (BeansException ex) {
         //如果创建bean异常 做一些清除工作
         //包括将bean从alreadyCreated，singletonsCurrentlyInCreation等容器中清除
		destroySingleton(beanName);
		throw ex;
        }}});
        //获取真正的bean，由于bean可以通过FactortBean或者工厂bean的方式创建
        //所以这个方法用于从Factorybean或者工厂bean获取要创建的真正的bean
	    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
		  }
        else if (mbd.isPrototype()) {
          //省略  原型模式的创建
            .............
        }else
        {
            //省略scope作用于的创建
             .............
        }
    }catch(BeansException ex){
        cleanupAfterBeanCreationFailure(beanName);
		throw ex;
    }
     //如果requiredType不为空，尝试将获取的bean转换成requiredType
        .............
}   
}
```

&ensp;&ensp;&ensp;&ensp;spring源码里面这段代码其实特别长，如果都搬下来，恐怕逻辑并不会这么清晰，这里主要分析主上上的核心方法，这些方法里面涵括了实例化一个bean的所有操作。

&ensp;&ensp;&ensp;&ensp;首先分析这个方法的几个参数：

- name：顾名思义就是bean的name

- requiredType：bean对象的类型

- args:创建bean实例构造函数所需要的参数列表，对应`getBean`第四个重载方法

- typeCheckOnly：bean实例是否包含一个类型检查

&ensp;&ensp;&ensp;&ensp;从当前案例来看，我们只使用了name属性，剩下的三个参数都是默认值，其中requiredType和args都是null，typeCheckOnly是false。从上面分析来看，实例化一个bean首先要获取构造函数进行初始化，然后设置属性值，我们来看这样一个情况，如果A中有一个属性为B b，B中也有一个属性A a，当实例化bean设置属性的属性，A会查找B的实例注入给属性b，如果B不存在会去创建B实例，当B实例设置属性a的时候回去查找A的实例，如果不存在则会创建A的实例，这个时候就会导致循环依赖的问题，Spring解决循环依赖是通过`ObjectFactory`作为中间对象提前暴露实例解决的。并且将这些提前暴露的实例放入到earlySingletonObjects中，当需要真正使用bean实例时，就会调用`ObjectFactory`的`getObject`获取，当然Spring只是通过这种方式解决了单例模式下的循环依赖问题，了解了这点下面看getBean也许会容易点。

&ensp;&ensp;&ensp;&ensp;梳理一下这个方法所干的事情:

- 获取真正的beanName，说白了就是将`FactoryBean`或者bean工厂类前面的&去掉

- 从单例容器中查找bean的缓存

- 如果缓存中存在，而且bean构造函数没有任何参数，直接创建bean实例返回

- 如果缓存不存在，将当前bean标记为创建状态，开始创建bean实例

- 从BeanDefinitions中查找bean声明，通过BeanDefinition创建bean返回

- 如果requiredType不为空，还需要将上面步骤生成的bean转换成requiredType返回

## 5、从IOC缓存中获取bean

&ensp;&ensp;&ensp;&ensp;接下里就看看各个方法所作的事情吧，首先看`getSingleton(beanName)`，直接定位到关键的方法实现：

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
         //从单例的IOC也就是hashMap中获取是否已经存在bean实例
		Object singletonObject = this.singletonObjects.get(beanName);
         //如果缓存不存在&&当前正常创建的单例对象就是该bean实例就进入创建方法
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            //为了线程安全将单例的IOC容器锁起来
			synchronized (this.singletonObjects) {
                 //从提前暴露容器中查找是否存在对应的ObjectFactory
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
                     //如果还是找不到就从bean工厂类的缓存中查找
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
                          //如果存在创建该实例的bean的工厂类 则通过工厂类创建bean
						singletonObject = singletonFactory.getObject();
                          //将创建的bean放入到提前暴露容器集合里
						this.earlySingletonObjects.put(beanName, singletonObject);
                          //因为已经存在了实例无需再通过工厂创建，
                          //将创建该实例的工厂从工厂让其中删除
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
        //返回从缓存中获取结果
		return (singletonObject != hNULL_OBJECT ? singletonObject : null);
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法的目的就是从单例缓存中获取bean的实例，首先从单例容器中获取，如果获取不到，再到提前暴露的容器里去查找，这个容器是`earlySingletonObjects`，也是一个ConcurrentHashMap，这个容器存在的意义就是为了解决循环依赖，所以将bean转换成`ObjectFactory`提前放入到容器中暴露出去，如果这个还找不到，就去创建bean的工厂类里面去查找，什么是bean的工厂类呢，看下这段配置就明了了:

```xml
 <bean id="bankPayService" class="com.yms.manager.serviceImpl.BankPayServiceImplFactory"              factory-method="getService"/>
```

&ensp;&ensp;&ensp;&ensp;bean的获取是通过`BankPayServiceImplFactory`的`getService`方法获取的，`BankPayServiceImplFactory`就叫做bean的工厂类，毫无疑问，如果是第一次使用bean实例，通过这个方法什么也获取不到，最终返回一个null，当在第二次使用bean实例的时候，直接执行到这里其实可以获取到第一次加载的bean的，对单例作用域的bean来说，无需再次实例化bean，极大的提高了spring的性能，这也是这一步为何如此关键的原因。

&ensp;&ensp;&ensp;&ensp;上面说了，如果是第一次获取某个bean实例，无论如何也无法从缓存中获取到bean的实例，但鉴于代码执行顺序原因，还是先把获取缓存的bean的情况分析下去，假如spring通过缓存拿到了bean实例对象，并不会直接返回对象，还需要执行方法 `getObjectForBeanInstance`:

```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

		// 如果bean实例不是FactoryBean 但name又是工厂的前缀引用&,直接抛异常
		if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}

		//如果beanInstance可能FactoryBean也可能是普通的bean
         //如果beanInstance是普通的bean或者仅仅是一个工厂实例 直接返回
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name))         {
			return beanInstance;
		}
		Object object = null;
		if (mbd == null) {
            //如果BeanDefinition为null，则从factoryBeanObjectCache查找实例
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			//经过上面判断已经确定beanInstance是FactoryBean
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// 如果bean是一个FactoryBean则缓存起来
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
             //这个就复杂了，主要用于当BeanDefinition的属性值
             //又是一个BeanDefinition的时候的解析赋值，后面aop还会分析
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法主要是获取真正的beanInstance，当从缓存里获取的bean实例是一个普通的bean或者是工厂bean的时候，直接返回，但如果是`FactroyBean`而且属性值又是一个`BeanDefinition`，即是一个合成bean的时候，需要调`BeanPostProcessor`的实现类去解析`BeanDefinition`最终生成一个bean实例，合成bean在AOP里面很常见，分析AOP的时候会详细说明，看到这里，其实就已经获取到了缓存中的bean实例对象了，不用过分关心最后一个方法，里面无非也就是将`eanDefinition`的属性值转换成bean实例的过程。

## 6、创建一个崭新的bean

&ensp;&ensp;&ensp;&ensp;经过上面两个方法，Spring已经可以从缓存中获取一个Bean的实例了，但是当第一次获取bean的时候，又该怎么处理呢，继续往下看，首先看一个if语句里面的方法`isPrototypeCurrentlyInCreation`:

```java
	protected boolean isPrototypeCurrentlyInCreation(String beanName) {
		Object curVal = this.prototypesCurrentlyInCreation.get();
		return (curVal != null &&
				(curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法很简单，就是在创建原型作用域的bean时首先去`prototypesCurrentlyInCreation`容器中检查是否存在beanName的bean正在创建，如果存在就抛出异常，`prototypesCurrentlyInCreation`这个集合每个线程都会保存一份，所以这里判断的是当前线程中名为beanName的bean是否正在创建，因为Spring只通过提前暴露的方式解决了单例bean的循环依赖，对于Prototype模式的bean只要当前线程中存在其他正在创建的相同的bean，就直接抛出异常，单例bean的创建只有一个线程，所以也可以用这段代码去校验，看一下prototypesCurrentlyInCreation的定义就清晰了。 

```java
ThreadLocal<Object> prototypesCurrentlyInCreation =
new NamedThreadLocal<Object>("Prototype beans currently in creation");
```

&ensp;&ensp;&ensp;&ensp;`ThreadLocal`会为每个线程生成一个`prototypesCurrentlyInCreation`集合，多个线程之间不受影响。

&ensp;&ensp;&ensp;&ensp;校验完bean的循环依赖后，还需要将bean标记成已经创建状态,标志着这bean已经被创建过，下次再来获取的时候，无需再次创建:

```java
	protected void markBeanAsCreated(String beanName) {
        //alreadyCreated也是一个hashMap
		this.alreadyCreated.add(beanName);
	}
```

&ensp;&ensp;&ensp;&ensp;此时bean实例创建的环境条件都已满足，可以开始创建bean了，首先需要根据beanName获取到`BeanDefinition`，下面的方法作用是获取到`BeanDefinition`，并将BeanDefinition转换为RootBeanDefinition:

```java
	protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
		//从mergedBeanDefinitions查找，该map里面存储的都是RootBeanDefinition
		RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
		if (mbd != null) {
			return mbd;
		}
        //如果根据beanName查询不到RootBeanDefinition，则转换成RootBeanDefinition
		return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
	}
```

&ensp;&ensp;&ensp;&ensp;这个时候就用到了IOC第一个大步骤中生成的BeanDefinition了，在上一篇中说了`BeanDefinition`里面主要存储了bean的类的所有信息，包括类的元素和属性信息，接下来就是解析`BeanDefinition`里面的数据，往bean实例里面填充就可以了，但是首先需要获取一个初始化后的bean实例，此时的bean应该仅仅是一个空的对象，属性什么的都没有赋值。因为是针对单例的bean分析的，接下来调用`getSingleton`方法，这个方法第一个参数是beanName，第二个是一个匿名内部类，bean对象实际就是由匿名内部类的实现方法createBean创建的：

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "'beanName' must not be null");
		synchronized (this.singletonObjects) {
            //再次尝试从单例容器中获取
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
                 //判断单例IOC容器是否已经被销毁
				if (this.singletonsCurrentlyInDestruction) {
                      //如果单例IOC容器都被销毁了 就直接终止bean的创建
					throw new BeanCreationNotAllowedException(beanName,"不能再已经销毁的容器中创建bean");
                }
                 //创建前检查 bean初始化的前置操作可以被覆盖，
                 //默认检查当前bean是否正在创建的bena的容器中
				beforeSingletonCreation(beanName);
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<Exception>();
				}
				try {
                      //核心方法最终会调用到上一个方法的匿名内部类的getObject
					singletonObject = singletonFactory.getObject();
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
                  //将创建的bean放入到单例IOC容器中
				addSingleton(beanName, singletonObject);
			}
			return (singletonObject != NULL_OBJECT ? singletonObject : null);
		}
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法首先是校验了单例IOC容器是否处于存活状态，如果已经被销毁直接抛出异常，不允许创建该bean，然后校验了当前bean是否已经放入到了正在创建的bean的集合中，也就是是否允许被创建，如果当前bean处于被允许被创建的集合中，调用参数`ObjectFactory`的getBean()方法获取bean对象，参数`singletonFactory`是一个匿名内部类，最终会执行核心方法5：`createBean(beanName, mbd, args)`这个方法去创建bean，等会就会看哪个方法，现在加入已经通过getBean获取到了bean实例对象，然后调用`addSingleton(beanName, singletonObject)`;将当前bean注册到单例IOC容器中。

&ensp;&ensp;&ensp;&ensp;截止到现在，终于可以看看创建一个bean的最核心的方法`createBean(beanName, mbd, args)`了，首先看看这个方法的参数，beanName是要获取bean的name，mbd是bean的BeanDefinition对象，args是动态指定的构造函数的参数集合，当然在这个测试案例里，args是null。createBean是一个抽象方法，由类`AbstractAutowireCapableBeanFactory`实现：

```java
     @Override
	protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args){
        //这一步主要是将配置得class转换成Class对象
		resolveBeanClass(mbd, beanName);
		try {
            //解析lookup-method和replaced-method方法
			mbd.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbd.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}
		try {
             //根据mbd校验是否是一个合成类，执行BeanPostProcessors处理器
            //这里可以创建一个代理对象
			Object bean = resolveBeforeInstantiation(beanName, mbd);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}
         //开始创建一个具体的，很单纯的bean，所谓单纯就是原生的bean
		Object beanInstance = doCreateBean(beanName, mbd, args);
		return beanInstance;
	}

```

&ensp;&ensp;&ensp;&ensp;这个方法里面其实也就是调用创建bean的方法，只不过在Spring里面创建bean有两种方式，一种就是原生bean的创建，另一种就是代理bean的创建，基于代理bean的创建的入口是在 实现了`BeanPostProcessor`的类中执行，这个在Spring AOP中会具体分析，这里针对测试案例而言时需要创建原生的bean，这样就进入到了`doCreateBean`方法中：

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
		//初始化bean的bean最终被封装到这个类里
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
             //因为是单例，需要从缓存中移除  之后再重新生成
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
             //这个方法会通过查找到的构造函数初始化一个纯净的属性未被赋值的bean
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

		//允许修改BeanDefinition
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				mbd.postProcessed = true;
			}
		}
		//校验bean是否使用了循环依赖
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
            //属性注入，为初始化的bean设置属性值
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
                 //这个地方会检查bean是否实现了一些Aware方法
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		catch (Throwable ex) {
		   throw ex;
		}
         //解决循环依赖问题，如果有循环依赖，重新获取bean对象，只是参数为false
         //表示再次获取时不允许有循环依赖问题，由于此时已经可以从缓存中获取到依赖的
         //bean对象，然后通过getObject()获取即可
		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
				}
			}
         // 最后将生成的bean实例注入到单例IOC容器中
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
		return exposedObject;
		}
```

&ensp;&ensp;&ensp;&ensp;这个方法真正完成了生成bean的操作，代码和复杂，但是最终完成了以下三件事情:

- `createBeanInstance`匹配一个合适的构造函数，通过反射创建一个bean的实例，此时的bean属性并未赋值

- `populateBean`是为生成的初始化bean设置属性值，用于setter注入，initializeBean是method注入

- 经过上两步，bean已经被实例化了，但是如果有循环依赖问题，属性值只是一个`ObjectFactroy`对象，并不是一个真正对象，所以这个时候需要再次调用`getSingleton`，从缓存中获取真正的bean。

&ensp;&ensp;&ensp;&ensp;就是这三个步骤最终完成生成bean的操作，前面看了这么多，其实都是铺垫，这三个步骤决定了创建bena的过程，跟着上面的总结，分别来看看这三步操作时如何执行的。

##       7、匹配一个构造函数

&ensp;&ensp;&ensp;&ensp;如果bean有默认的无参构造函数，可以直接通过反射初始化一个bean对象，但是如果需要使用指定的构造函数初始化一个bean，就需要去查找合适的构造函数，然后完成初始化。Spring在设置一个构造函数参数的时候，允许使用name，type，index表示一个参数，使用ref和value表示参数值，其中name和type可以一起使用，参数值ref和value只能存在一个，看下面的方法:

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
		//将class转换成Class，其实上面已经转换过了，直接就会获取到，不会再次加载
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
         //如果Class不允许被实例化 直接抛出异常
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
         //如果配置了factory-method，则从factory-method方法中获取bean
		if (mbd.getFactoryMethodName() != null)  {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}
		//标识bean是否被创建并且解析过
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}
		//获取beanClass的所有构造函数
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
             //查找合适的构造函数然后初始化bean
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		//直接使用默认无参构造函数初始化bean
		return instantiateBean(beanName, mbd);
	}
```

其实`createBeanInstance`并没有完成初始化bean操作，仅仅是针对不同情况，使用了不同方式的初始化，比如如果bean配置了factory-method，就是使用配置的工厂方法去创建一个bean，如果beanClass构造函数列表不为空，则先去匹配到一个构造函数，再去初始化，如果beanClass有默认的无参构造函数，则直接进行初始化即可，这里来分析使用匹配的构造函数初始化bena的过程，测试案例中最终也会执行`autowireConstructor(beanName, mbd, ctors, args)`:

```java
	protected BeanWrapper autowireConstructor(
			String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs) {
		return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
	}
```

&ensp;&ensp;&ensp;&ensp;从上面代码看来，初始化bean的操作被委托给了`ConstructorResolver`的`autowireConstructor`方法,说实话，这段代码实现太长了， 都写下来恐怕真的很复杂，分析之前还是先简单的介绍下这个方法的作用：

- 根据参数个数以及参数的name，type或者index匹配一个构造函数 初始化bean，这里读取构造函数参数名是使用asm读取类的字节码文件获取到的。

- 获取参数的真实值，在生成`BeanDefinition`时候，ref的值会被保存为`RuntimeBeanReference`，value的值会被保存为`TypedStringValue`，这里会根据不同的类型值使用不同的解析方式。

&ensp;&ensp;&ensp;&ensp;这里还是精简后的代码，只分析核心部分:

```java
	public BeanWrapper autowireConstructor(
			final String beanName, final RootBeanDefinition mbd, Constructor[] chosenCtors, final Object[] explicitArgs) {
			//省略代码....
         //最终匹配到的要使用的构造函数
         Constructor constructorToUse = null;
		ArgumentsHolder argsHolderToUse = null;
         //匹配到的构造函数所使用的参数列表
		Object[] argsToUse = null;
        //explicitArgs是在getBean(benaName,args)指定了构造函数参数列表
		if (explicitArgs != null) {
			argsToUse = explicitArgs;
		}
        //如果未指定构造函数的参数列表，要根据配置文件去匹配合适的构造函数的参数列表
        else 
        {
            //尝试从缓存中获取 
            ..........
        }
        //如果构造函数参数列表未指定  也无法从缓存中获取 读取配置文件配置
        if (constructorToUse == null) {
			boolean autowiring = (chosenCtors != null ||
					mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
			ConstructorArgumentValues resolvedValues = null;
			int minNrOfArgs;
			if (explicitArgs != null) {
				minNrOfArgs = explicitArgs.length;
			}
			else {
                 //从BeanDefinition中获取 也就是从配置文件中获取
				ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
                 //由于构造函数参数可能需要转换成真实值，用这个类接收参数值
				resolvedValues = new ConstructorArgumentValues();
                 //将参数值放入到ConstructorArgumentValues中并返回参数个数
                 //这个方法会将RuntimeBeanReference，TypedStringValue等中间值解析成真实值
                 //等会需要看这个方法
				minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
			}
          	//如果指定了构造函数，则从指定的构造函数中查找合适的，这里为null
			Constructor[] candidates = chosenCtors;
			if (candidates == null) 
            {
				Class beanClass = mbd.getBeanClass();
				try {
                     //获取bean实例的所有构造函数
					candidates = (mbd.isNonPublicAccessAllowed() ?
							beanClass.getDeclaredConstructors() : beanClass.getConstructors());
				}
				catch (Throwable ex) {
                    //bean不能被实例化  无法获取构造函数 抛出异常
			      throw ex；
				}
			}
             //根据构造函数参数个数进行排序
			AutowireUtils.sortConstructors(candidates);
			int minTypeDiffWeight = Integer.MAX_VALUE;
			Set<Constructor> ambiguousConstructors = null;
			List<Exception> causes = null;
                 //遍历bean的所有构造函数  最终找到一个合适的构造函数
            	for (int i = 0; i < candidates.length; i++) {
				Constructor<?> candidate = candidates[i];
				Class[] paramTypes = candidate.getParameterTypes();
				if (constructorToUse != null && argsToUse.length > paramTypes.length) {
                     //因为前面已经对构造函数参数根据参数个数进行排序，所以如果当前的参数
                     //个数都不满足，后面的构造函数参数个数肯定不满足  直接中断循环即可
					break;
				}
                  //如果
				if (paramTypes.length < minNrOfArgs) {
					continue;
				}
				ArgumentsHolder argsHolder;
                  //下面这个执行体里主要是通过asm获取构造函数的方法参数名
                  //根据参数名和参数类型去检索出一个合适的构造函数
				if (resolvedValues != null) {
					try {
						String[] paramNames = null;
						if (constructorPropertiesAnnotationAvailable) {
paramNames = ConstructorPropertiesChecker.evaluateAnnotation(candidate, paramTypes.length);
						}
						if (paramNames == null) {
ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
							if (pnd != null) {
								paramNames = pnd.getParameterNames(candidate);
							}
						}
argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames, candidate, autowiring);
					}
					catch (UnsatisfiedDependencyException ex) {
					 throw ex;
				}
				else {
					//根据传入的参数列表 获取一个构造函数
					if (paramTypes.length != explicitArgs.length) {
						continue;
					}
					argsHolder = new ArgumentsHolder(explicitArgs);
				}
				int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
						argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
				// 如果匹配到了就选择这个构造函数 并将参数值赋予到
				if (typeDiffWeight < minTypeDiffWeight) {
					constructorToUse = candidate;
					argsHolderToUse = argsHolder;
					argsToUse = argsHolder.arguments;
					minTypeDiffWeight = typeDiffWeight;
					ambiguousConstructors = null;
				}
				else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
					if (ambiguousConstructors == null) {
						ambiguousConstructors = new LinkedHashSet<Constructor>();
						ambiguousConstructors.add(constructorToUse);
					}
					ambiguousConstructors.add(candidate);
				}
			}
            //省略一些代码......
	       Object beanInstance;
			if (System.getSecurityManager() != null) {
				.........
			}
			else {
                 //最终根据选择构造函数和参数列表 初始化一个bean实例
				beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
						mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
			}
			bw.setWrappedInstance(beanInstance);
			return bw;
        }
			}
```

&ensp;&ensp;&ensp;&ensp;这段代码很长，真想看明白还是建议通过断点调试的方式一点一点看里面的内容，针对这里面的很多方法不在具体分析，这个代码其实就完成了一个功能，匹配构造函数，只不过这里分了三种情况，如果指定了构造参数`explicitArgs`，则使用，如果未指定则从缓存中读取，如果还未读取到则从BeanDefinition获取，最终获取到要使用的构造函数参数，然后获取bean的所有构造函数，根据参数name和type去匹配一个构造函数，最终初始化一个bean。在创建`BeanDefinition`的时候，属性值或者参数值是通过`RuntimeBeanReference`等中间类表示的，在下面的方法里就会将这些中间类解析成真正的值:

```java
public Object resolveValueIfNecessary(Object argName, Object value) {
		// We must check each value to see whether it requires a runtime reference
		// to another bean to be resolved.
		if (value instanceof RuntimeBeanReference) {
			RuntimeBeanReference ref = (RuntimeBeanReference) value;
			return resolveReference(argName, ref);
		}
		else if (value instanceof RuntimeBeanNameReference) {
			String refName = ((RuntimeBeanNameReference) value).getBeanName();
			refName = String.valueOf(evaluate(refName));
			if (!this.beanFactory.containsBean(refName)) {
				throw new BeanDefinitionStoreException(
						"Invalid bean name '" + refName + "' in bean reference for " + argName);
			}
			return refName;
		}
		else if (value instanceof BeanDefinitionHolder) {
			// Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
			BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
			return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
		}
		else if (value instanceof BeanDefinition) {
			// Resolve plain BeanDefinition, without contained name: use dummy name.
			BeanDefinition bd = (BeanDefinition) value;
			return resolveInnerBean(argName, "(inner bean)", bd);
		}
		  .........
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法会将中间类最终转换成实际的值，这里面有很多类型，可以支持更复杂的`BeanDefinition`参数的解析。

## 8、完成bean的属性注入

&ensp;&ensp;&ensp;&ensp;经过上面分析，Spring会为bean匹配一个构造函数，然后得到初始化后的bean，此时的bean只是一个很"单纯"的bean，所谓单纯也就是bean并没有完成实例化操作，也就是没有把相关属性值注入到这个bean的属性中，下面的方法就是完成属性注入的，属性注入一般有两种方式，一种是通过name注入也就是`autowireByName`，还有一种是根据类型注入，也就是`autowireByType`。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
PropertyValues pvs = mbd.getPropertyValues();
		if (bw == null) {
			if (!pvs.isEmpty()) {
                //如果bw为空 而且mbd里面有配置的属性，代表初始化失败，直接抛出异常
			}
			else {
				//初始化失败  没有要注入的属性  继续直接返回
				return;
			}
		}
    //执行 BeanPostProcessor 如果有的话，这个是在bean进行属性设置之前对bean注入
    //比如通过注解的方式进行属性注入就是在这里完成的
     if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}
     		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// 通过属性name注入值
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// 通过属性的classType为属性注入值
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}
        //这部分也执行BeanPostProcessor
    	if (hasInstAwareBpps || needsDepCheck) {
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);

		}
         //最终在这个方法里面完成属性的设置的
		applyPropertyValues(beanName, mbd, bw, pvs);
}
```

&ensp;&ensp;&ensp;&ensp;这个方法其实就是从BeanDefinition中获取配置的属性值，首先会通过`BeanPostProcessor`实现的类解析通过注解设置的属性，然后再解析通过配置文件配置的属性，注入属性有两种方式`autowireByName`和`autowireByType`，这个过程其实就是将`BeanDefinition`引用的中间值转换成真实值的过程，然后通过`applyPropertyValues`使用真实值对初始化的bean进行实例化。思路其实很简单，spring将通过注解的方式注入属性的逻辑放入到了`BeanPostProcessor`中，复杂的逻辑其实在`BeanPostProcessor`中，哪个实现比起分析的基于xml配置的实现，封装性要更强一点，但是最终达到的效果是一样的，理解了基于xml配置的属性注入方式，基于注解的配置只要耐心的看，也很容易理解，属性注入的代码其实没什么好分析的。主要看下`autowireByType`和`autowireByType`，这两个方法如果是基于xml的需要配置`autowire`属性，分别对应byName和byType，但是还是要对比看看spring是如何实现的，首先`autowireByType`：

```java
	protected void autowireByName(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {  
        //获取注解的所有属性name
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
             //判断容器中是否包含name的实例
			if (containsBean(propertyName)) {
                 //如果直接包含 直接获取该实例即可
				Object bean = getBean(propertyName);
                 //将实例添加到bean的属性集合里 等到注入
				pvs.add(propertyName, bean);
                 //由于创建当前bena的时候，属性实例也被附带创建了，所以要注册到IOC容器中
				registerDependentBean(propertyName, beanName);
			}
			else {
		        //IOC容器中没有对应name的属性值
			}
		}
	}
```

&ensp;&ensp;&ensp;&ensp;基于name去获取bean实例从代码上看非常简单，仅仅是通过name从IOC中获取属性名为`propertyName`的实例。如果赵傲则放入到bean实例的属性参数集合中，等到被注入即可。与`autowireByType`相比，`autowireByType`可就复杂多了，复杂的原因就是要解析依赖项的过程。

```java
protected void autowireByType(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
		Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
			try {
				PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
			    //如果属性类型为object 直接忽略就行
				if (!Object.class.equals(pd.getPropertyType())) {
                       //获取 setter 方法（write method）的参数信息，比如参数在参数列表中的
                      //位置，参数类型，以及该参数所归属的方法等信息
MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass());
//创建属性依赖描述对象 这里面包含了字段类型 字段名  已经是否必输等信息
DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
//解析属性依赖。获取bean对象  这里面比较复杂 是主要查找过程
Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
					if (autowiredArgument != null) {
						pvs.add(propertyName, autowiredArgument);
					}
					for (String autowiredBeanName : autowiredBeanNames) {
						registerDependentBean(autowiredBeanName, beanName);
					}
					autowiredBeanNames.clear();
				}
			}
			catch (BeansException ex) {
				throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
			}
		}
	}
```

&ensp;&ensp;&ensp;&ensp;如上所示，`autowireByType` 的代码本身并不复杂。和 `autowireByName` 一样，`autowireByType` 首先也是获取非简单类型属性的名称。然后再根据属性名获取属性描述符，并由属性描述符获取方法参数对象 `MethodParameter`，随后再根据 `MethodParameter` 对象获取依赖描述符对象，在获取到依赖描述符对象后，再根据依赖描述符解析出合适的依赖。最后将解析出的结果存入属性列表 pvs 中即可，核心的解析依赖对象在方法`resolveDependency`中，这里就不展开分析了，感兴趣的可以调试看下。

## 9、Aware和init-method方法

&ensp;&ensp;&ensp;&ensp;经过上面的分析，Spring通过`autowireConstructor`方法完成bean的初始化，通过`populateBean`完成bean的属性注入(实例化)，这只是最基本的bean的创建过程，在Spring中还有一些与bean生命周期相关的Aware接口，以及实例化后调用`init-method`指定的方法的过程，回头看下`populateBean`之后的代码，这段代码在`doCreateBean`中：

```java
try {
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
                 //这个方法会检查bean是否实现了Aware接口 是否指定了init-method方法
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
```

查看initializeBean方法

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				@Override
				public Object run() {
                      //检查是否实现了Aware接口
					invokeAwareMethods(beanName, bean);
					return null;
				}
			}, getAccessControlContext());
		}
		else {
              //检查是否实现了Aware接口
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
            //执行BeanPostProcessor的postProcessBeforeInitialization
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
             //查找是否指定了init-method方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}

		if (mbd == null || !mbd.isSynthetic()) {
              //执行BeanPostProcessor的postProcessAfterInitialization
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
		return wrappedBean;
	}
```

&ensp;&ensp;&ensp;&ensp;在这个方法中，主要做的事情可以概括为以下四步：

- 首先会调用`invokeAwareMethods`检查bean是否实现了Aware接口，如果实现了就调用Aware接口，

- 调用`applyBeanPostProcessorsBeforeInitialization`方法，这个方法也是在bean创建的生命周期中，是bean在属性填充完毕之后在bean初始化方法调用之前执行，实际上是调用的`BeanPostProcessor`的`postProcessBeforeInitialization`方法

- 调用`invokeInitMethods`去检查bean是否指定了`init-method`方法，如果指定了就执行

- 调用`applyBeanPostProcessorsAfterInitialization`，对应的是调用`BeanPostProcessor`的`postProcessAfterInitialization`方法

&ensp;&ensp;&ensp;&ensp;接下来主要看看`invokeAwareMethods`和`invokeInitMethods`这两个方法：

```java
	private void invokeAwareMethods(final String beanName, final Object bean) {
        //检查bena是否实现了对应的Aware接口
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
```

&ensp;&ensp;&ensp;&ensp; 下面的方法是检查bean是否指定了`init-method`方法

```java
	protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
			throws Throwable {
         //判断bean的属性是否由BeanFactoryAware或者BeanFactory等创建
		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						@Override
						public Object run() throws Exception {
							((InitializingBean) bean).afterPropertiesSet();
							return null;
						}
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
                 //调用bean的afterPropertiesSet再次解析bean的属性
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null) {
             //查看是否指定了init-method
			String initMethodName = mbd.getInitMethodName();
			if (initMethodName != null && !(isInitializingBean &&               "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
                //如果指定了就执行init-method方法
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}

```
## 10、解决循环依赖

&ensp;&ensp;&ensp;&ensp;前面已经简单说过Spring是如何解决循环依赖问题，由于上面主要是分析Spring源码的过程，可能说的不是那么清晰，我觉得有必要将这部分内容单独拿出来说一说。

&ensp;&ensp;&ensp;&ensp;经过前面的步骤，已经可以获取到一个经过初始化的bean实例，而且完成了bean的属性注入，但是前面还有一个问题没有解决，如果在单例IOC中有循环依赖，暂时是用`ObjectFactory`中间类代替的，所以还要通过`ObjectFactory`的`getObject`方法获取真正的bean，此时所有的bean已经被创建完成，所以通过`getBean`可以获取到真实的bean实例对象，再看看下面的代码，这段代码也是在`doCreateBean`中：

```java
          //解决循环依赖问题，如果有循环依赖，重新获取bean对象，只是参数为false
         //表示再次获取时不允许有循环依赖问题，由于此时已经可以从缓存中获取到依赖的
         //bean对象，然后通过getObject()获取即可
		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
				}
			}
```

&ensp;&ensp;&ensp;&ensp;Spring解决循环依赖是通过`ObjectFactory`中间类实现的，当然通过这个类只能解决单例IOC的属性循环依赖的问题，对于原型模式和构造器的循环以来，spring至今也没有解决方法，首先来回顾一下Spring获取一个bean的简单过程：

- 首先从`singletonObjects`获取，也就是单例IOC容器中
- 然后从`earlySingletonObjects`中获取，也就是通过`ObjectFactory`实现的提前曝光的容器
- 最后从`singletonFactories`获取，也就是实例化bean的实例工厂
- 如果都获取不到，则新创建一个bean对象，也就说走上面分析的流程

&ensp;&ensp;&ensp;&ensp;从缓存中获取的bean的过程，网上很多都叫三级缓存，前面三个步骤对应了1，2，3级缓存。接下来举一个案例分析这个过程，假如有bean的依赖关系为：A->B->C->A，当然这些都是基于属性依赖的，当A执行到`populateBean`方法实现属性注入的时候，会先去获取B实例，然后B执行`populateBean`会获取C实例，C执行到`populateBean`获取查找A实例，此时A实例正在被创建，又会循环上述过程，产生了循环依赖问题。Spring获取`getBean()`最终调用下面简化后的方法:

```java
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
		final String beanName = transformedBeanName(name);
		Object bean;
		//关注点1
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		else {
			try {
				if (mbd.isSingleton()) {
                     //关注点2
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								throw ex;
							}
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
            ..............
            }
		return (T) bean;
	}

```

​       当A去查找bean的实例的时候，会调用上面的`doGetBean`方法获取，这个方法里面有两个关注点，分别是两个重载方法`getSingleton`，首先当A执行`populateBean`查找B的实例时调用第一个重载：

```java
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //从一级缓存中获取
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
                 //从二级缓存中获取
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
                          //从三级缓存中获取
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法是从三级缓存中查找bean，第一级缓存`singletonObjects`里面放置的是实例化好的单例对象。第二级`earlySingletonObjects`里面存放的是提前曝光的单例对象（没有完全装配好）。第三级`singletonFactories`里面存放的是要被实例化的对象的对象工厂，由于B第一次获取还没有被创建，所以一级缓存`singletonObjects`获取结果肯定为null，再看看看看进入二级缓存中的条件`isSingletonCurrentlyInCreation(beanName)`：

```java
	public boolean isSingletonCurrentlyInCreation(String beanName) {
         //在这里表示bean是否正在创建的过程，此时B 尚未在创建中，所以会返回false
		return this.singletonsCurrentlyInCreation.contains(beanName);
	}
```

&ensp;&ensp;&ensp;&ensp;上面的步骤中并没有任何操作往`isSingletonCurrentlyInCreation`中加入B的beanName的操作，所以压根不会进入二级缓存，直接就返回null了，然后就判断bean是否时单例的，如果时调用`getSingleton(String beanName, ObjectFactory objectFactory)`,此时objectFactory时一个匿名内部类，实例B的获取是通过内部类的`createBean`获取的，这个是我们关注点2 ：

```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "'beanName' must not be null");
		synchronized (this.singletonObjects) {
             //从一级缓存中获取
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,"不能从销毁的bean中创建");
				}
                 //在这里将B的beanName添加到isSingletonCurrentlyInCreation
				beforeSingletonCreation(beanName);
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<Exception>();
				}
				try {
                      //最终调用匿名内部类创建bean
					singletonObject = singletonFactory.getObject();
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
				addSingleton(beanName, singletonObject);
			}
			return (singletonObject != NULL_OBJECT ? singletonObject : null);
		}
	}
```

&ensp;&ensp;&ensp;&ensp;这个方法首先会从一级缓存中查找B，很明显，查到的结果为null，然后调用`beforeSingletonCreation(beanName)`，将B的beanName添加到`singletonsCurrentlyInCreation`中，也就是关注点1中无法进入二级缓存的那个集合校验：

```java
	protected void beforeSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) &&
				!this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
	}
```

&ensp;&ensp;&ensp;&ensp;紧接着就会调用`singletonFactory.getObject()`创建名，也就是通过匿名内部类的`createBean`方法创建，前面分析过，创建bean最终会调用`doCreateBean`方法，这个方法简化了， 只看最核心的关注点3：

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
    ...........代码省略...........
   //关注点3
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isDebugEnabled()) {
         logger.debug("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      addSingletonFactory(beanName, new ObjectFactory<Object>() {
         @Override
         public Object getObject() throws BeansException {
            return getEarlyBeanReference(beanName, mbd, bean);
         }
      });
   }
   //初始化和实例化bean
   Object exposedObject = bean;
   try {
      populateBean(beanName, mbd, instanceWrapper);
      if (exposedObject != null) {
         exposedObject = initializeBean(beanName, exposedObject, mbd);
      }
   }
   catch (Throwable ex) {
     throw ex;
   }
   if (earlySingletonExposure) {
      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
         if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
         }
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  actualDependentBeans.add(dependentBean);
               }
            }
         }
      }
   }
  ...........代码省略...........
   return exposedObject;
}
```

&ensp;&ensp;&ensp;&ensp;`createBeanInstance`利用反射创建了对象，下面我们看看关注点3` earlySingletonExposure`属性值的判断，其中有一个判断点就是`isSingletonCurrentlyInCreation(beanName)`

```java
	public boolean isSingletonCurrentlyInCreation(String beanName) {
		return this.singletonsCurrentlyInCreation.contains(beanName);
	}
```

&ensp;&ensp;&ensp;&ensp;发现使用的是`singletonsCurrentlyInCreation`这个集合，在上面的步骤中将的B的BeanName已经填充进去了，所以可以查到，而且在初始化bean的时候，还会判断检查bean是否有循环依赖，而且是否允许循环依赖，这里的ABC形成了循环依赖，所以最终`earlySingletonExposure`结合其他的条件综合判断为true,进行下面的流程`addSingletonFactory`,这里是为这个Bean添加`ObjectFactory`,这个`BeanName(A)`对应的对象工厂，他的`getObject`方法的实现是通过`getEarlyBeanReference`这个方法实现的。首先我们看下`addSingletonFactory`的实现

```java
	protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
```

&ensp;&ensp;&ensp;&ensp;往三级缓存`singletonFactories`存放数据，清除二级缓存中beanName对应的数据。这里有个很重要的点，是往三级缓存里面存入了值，这是Spring处理循环依赖的核心点。`getEarlyBeanReference`这个方法是`getObject`的实现，可以简单认为是返回了一个为填充完毕的A的对象实例。设置完三级缓存后，就开始了填充A对象属性的过程。

&ensp;&ensp;&ensp;&ensp;上面理清之后整体来分析以下ABC的初始化流程，当设置A的属性时，发现需要B类型的Bean，于是继续调用`getBean`方法创建，这次的流程和上面A的完全一致，然后到了填充C类型的Bean的过程，同样的调用getBean(C)来执行，同样到了填充属性A的时候，调用了getBean(A),我们从这里继续说，调用了doGetBean中的`Object sharedInstance = getSingleton(beanName),还是关注点1的代码，但是处理逻辑完全不一样了。

```java
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //从一级缓存中获取
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
                 //从二级缓存中获取 此时二级缓存中应该也获取不到 
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
                          //从三级缓存中获取  此时可以获取到 A 的实例，虽然属性并不太完整
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```

&ensp;&ensp;&ensp;&ensp;还是从singletonObjects获取对象获取不到，因为A是在`singletonsCurrentlyInCreation`这个Set中，所以进入了下面的逻辑，从二级缓存`earlySingletonObjects`中取，还是没有查到，然后从三级缓存`singletonFactories`找到对应的对象工厂调用`getObject`方法获取未完全填充完毕的A的实例对象，然后删除三级缓存的数据，填充二级缓存的数据，返回这个对象A。C依赖A的实例填充完毕了，虽然这个A是不完整的。不管怎么样C式填充完了，就可以将C放到一级缓存`singletonObjects`同时清理二级和三级缓存的数据。同样的流程B依赖的C填充好了，B也就填充好了，同理A依赖的B填充好了，A也就填充好了。Spring就是通过这种方式来解决循环引用的。

## 11、总结

&ensp;&ensp;&ensp;&ensp;通过上面的代码分析，Spring通过`BeanFactory`的`getBean()`方法获取bean实例的大致流程为：

- 从缓存中获取bean实例（三级缓存  解决循环依赖问题）

- 如果bean没有被创建过，执行创建bean的流程

- `autowireConstructor`匹配构造函数，通过反射创建一个bean实例

- 通过`populateBean`完成bean的属性注入

- 通过`initializeBean`检查bean配置得初始化方法和Aware接口

- 将创建得bean加入到IOC容器中

&ensp;&ensp;&ensp;&ensp;实际上，如果不考虑注解的话 ，Spring解析配置文件并完成bean的创建过程，就是上面的那部分代码，没有太多复杂的算法和抽象结构，很容易理解。Spring把很多解析注解的逻辑放入到了`BeanPostProcessor`中执行并通过这些实现类控制了bean的生命周期，在每个生成BeanDefinition或者getBean()的关键方法中，总会在方法前或后执行这些接口方法，这些接口最终会调用到封装好的功能，比如通过注解声明类和属性注入还有AOP的方法拦截。如果想了解这部分的实现，可以看下BeanPostProcessor接口的实现类，主要功能在那里封装的，内容比较抽象，但还是很值得看的。到此为止，Spring IOC的原理基本上就分析完了，接下来会分析Spring AOP的部分。
