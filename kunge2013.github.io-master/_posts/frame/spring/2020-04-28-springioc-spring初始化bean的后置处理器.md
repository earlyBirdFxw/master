---
layout: post
title:  "spring ioc bean的后置处理器"
date:   2020-04-28 00:06:05
categories: spring
tags: spring 源码
---

##### spring ioc bean的后置处理



---

##### 1.springioc 中有两个接口都可以被现实下，分别是 BeanFactoryPostProcessor和BeanPostProcessor


-   1.BeanFactoryPostProcessor和BeanPostProcessor有什么区别呢?

 a.BeanFactoryPostProcessor用于对beanFactory的对象的增强，修改beanFactory的相关属性
   beanFactory可以直接注册bean,修改属性
   
 b.BeanPostProcessor用于对实例bean创建后的增强，比如在bean实例创建前后，可以修改bean实例的属性值， bean的生命周期主要是围绕BeanPostProcessor来实现的
 
   beanPostProcessor可以偷梁换柱，修改bean的属性信息！
   
 以下是 BeanPostProcessor和BeanFactoryPostProcessor的部分源码实现
 	
 	
	public class InitBeanPostProcessor implements BeanPostProcessor {
	
		@Override
		public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
			// TODO Auto-generated method stub
			System.out.println(" postProcessBeforeInitialization ===" + beanName);
			if (bean instanceof String) {
				bean = "String 的对象被修改了值";
				return bean;
			}
			return BeanPostProcessor.super.postProcessBeforeInitialization(bean, beanName);
		}
		
		
		@Override
		public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
			// TODO Auto-generated method stub
			System.out.println(" postProcessAfterInitialization ===" + beanName);
			return BeanPostProcessor.super.postProcessAfterInitialization(bean, beanName);
		}
		
		
		
	}
	

	package org.kframe.springioc.annotations.processor;
	
	import org.springframework.beans.BeansException;
	import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
	import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
	import org.springframework.stereotype.Component;
	
	@Component
	public class SelfBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
	
		@Override
		public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
			System.out.println(beanFactory);
			beanFactory.registerSingleton("String", new String("11111"));
		}
	
	}



	package org.kframe.springioc.annotations;
	
	import org.springframework.context.annotation.AnnotationConfigApplicationContext;
	
	public class AnnotationApplication {
		
		private static AnnotationConfigApplicationContext context;
		
		public static void main(String[] args) throws Exception {
			System.out.println("=======================begin  AnnotationConfigApplicationContext=======================================");
			context = new AnnotationConfigApplicationContext("org.kframe.springioc.annotations");
			System.out.println("String==== 代码手动注入后的String ：" +  context.getBean("String"));
			System.out.println("config==== name 被改编后的值 ：" +  context.getBean(Config.class).getName());
		}
		
	}


---
	
##### 2.spring bean 对象注解初始化 后置处理器 InitDestroyAnnotationBeanPostProcessor

---

-   1.InitDestroyAnnotationBeanPostProcessor后置处理器的作用?
	InitDestroyAnnotationBeanPostProcessor 后置处理器主要用于，实例化bean后解析 PostConstruct 和 PreDestroy 注解，如果当前的bean 
	配置了PostConstruct 或 PreDestroy，那么 会通过反射的方式获取到当前bean的方法，然后在执行PostConstruct和PreDestroy对应的方法
	
		public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
			LifecycleMetadata metadata = findLifecycleMetadata(beanType);
			metadata.checkConfigMembers(beanDefinition);
		}
		/**
		 * 	查找或者初始化 当前bean 实例的初始化方法或者销毁方法 （PostConstruct 或者  PreDestroy）
		 * @param clazz
		 * @return
		 */
		private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {
			if (this.lifecycleMetadataCache == null) {
				// Happens after deserialization, during destruction...
				return buildLifecycleMetadata(clazz);
			}
			// Quick check on the concurrent map first, with minimal locking.
			LifecycleMetadata metadata = this.lifecycleMetadataCache.get(clazz);
			if (metadata == null) {
				synchronized (this.lifecycleMetadataCache) {
					metadata = this.lifecycleMetadataCache.get(clazz);
					if (metadata == null) {
				// 获取是由引用了PostConstruct或者  PreDestroy的相关注解 ，也就是是否自定义了对bean 的自定义的方法
						metadata = buildLifecycleMetadata(clazz);
						this.lifecycleMetadataCache.put(clazz, metadata);
					}
					return metadata;
				}
			}
			return metadata;
		}
		/**
		 * 获取当前bean 实例中是否包含 PostConstruct 或者  PreDestroy的相关注解
		 * @param clazz
		 * @return
		 */
		private LifecycleMetadata buildLifecycleMetadata(final Class<?> clazz) {
			if (!AnnotationUtils.isCandidateClass(clazz, Arrays.asList(this.initAnnotationType, this.destroyAnnotationType))) {
				return this.emptyLifecycleMetadata;
			}
			List<LifecycleElement> initMethods = new ArrayList<>();
			List<LifecycleElement> destroyMethods = new ArrayList<>();
			Class<?> targetClass = clazz;
			do {
				final List<LifecycleElement> currInitMethods = new ArrayList<>();
				final List<LifecycleElement> currDestroyMethods = new ArrayList<>();
				// 此处主要通过反射方式获取方法体中包不包含 初始化注解，销毁注解  
				ReflectionUtils.doWithLocalMethods(targetClass, method -> {
					if (this.initAnnotationType != null && method.isAnnotationPresent(this.initAnnotationType)) {
						LifecycleElement element = new LifecycleElement(method);
						currInitMethods.add(element);
						if (logger.isTraceEnabled()) {
							logger.trace("Found init method on class [" + clazz.getName() + "]: " + method);
						}
					}
					if (this.destroyAnnotationType != null && method.isAnnotationPresent(this.destroyAnnotationType)) {
						currDestroyMethods.add(new LifecycleElement(method));
						if (logger.isTraceEnabled()) {
							logger.trace("Found destroy method on class [" + clazz.getName() + "]: " + method);
						}
					}
				});
				initMethods.addAll(0, currInitMethods);
				destroyMethods.addAll(currDestroyMethods);
				targetClass = targetClass.getSuperclass();
			}
			while (targetClass != null && targetClass != Object.class);
			return (initMethods.isEmpty() && destroyMethods.isEmpty() ? this.emptyLifecycleMetadata :
					new LifecycleMetadata(clazz, initMethods, destroyMethods));
		}	

	


--- 	
	
-   2.初始化参数， 该后置处理器在org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory # applyMergedBeanDefinitionPostProcessors中调用初始化参数

		/**
		 *  	对指定的bean定义应用MergedBeanDefinitionPostProcessors，调用它们的postProcessMergedBeanDefinition方法。
		 * Apply MergedBeanDefinitionPostProcessors to the specified bean definition,
		 * invoking their {@code postProcessMergedBeanDefinition} methods.
		 * @param mbd the merged bean definition for the bean
		 * @param beanType the actual type of the managed bean instance
		 * @param beanName the name of the bean
		 * @see MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition
		 */
		protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof MergedBeanDefinitionPostProcessor) {
					MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
					bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
				}
			}
		}


--- 

-   3.执行, 该后置处理器在org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory # applyMergedBeanDefinitionPostProcessors中调用初始化参数
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
	#applyBeanPostProcessorsBeforeInitialization 调用执行后置处理器,调用位置如下
	
-  a.上层调用栈

		/**
			 * 执行后置处理器
			 */
			@Override
			public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
					throws BeansException {
		
				Object result = existingBean;
				for (BeanPostProcessor processor : getBeanPostProcessors()) {
					Object current = processor.postProcessBeforeInitialization(result, beanName);
					if (current == null) {
						return result;
					}
					result = current;
				}
				return result;
			}
		
-   b.执行后置处理器并初始化bean的属性值	，初始化代码逻辑如下
	
	
		/**
		 * 	执行后置处理器
		 * 	初始化bean的属性
		 * 
		 */
		@Override
		public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
			LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
			try {
				metadata.invokeInitMethods(bean, beanName);
			}
			catch (InvocationTargetException ex) {
				throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());
			}
			catch (Throwable ex) {
				throw new BeanCreationException(beanName, "Failed to invoke init method", ex);
			}
			return bean;
		}
	

##### 3.spring 动态代理bean 的创建

-   a.spring 动态代理bean 是在第8次后置处理器中生成 , 代码如下


				/**
				 * XXX 第八次调用后置处理器
				 *	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization
				 *	调用的是BeanPostProcessor --> postProcessAfterInitialization bean初始化之后执行的方法(处理AOP)
				 */
				@Override
				public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
						throws BeansException {
					Object result = existingBean;
					for (BeanPostProcessor processor : getBeanPostProcessors()) {
						/**
						 * 创建动态代理对象
						 */
						Object current = processor.postProcessAfterInitialization(result, beanName); //见 b的代码块
						if (current == null) {
							return result;
						}
						result = current;
					}
					return result;
				}



-   b.具体后置处理器是通过 org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator 的 postProcessAfterInitialization(@Nullable Object bean, String beanName)生成
	
			/**
			 * Create a proxy with the configured interceptors if the bean is
			 * identified as one to proxy by the subclass.
			 * @see #getAdvicesAndAdvisorsForBean
			 * XXX  生成代理对象
			 */
			@Override
			public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
				if (bean != null) {
					Object cacheKey = getCacheKey(bean.getClass(), beanName);
					if (this.earlyProxyReferences.remove(cacheKey) != bean) {
						return wrapIfNecessary(bean, beanName, cacheKey);
					}
				}
				return bean;
			}
		
		/**
		 * XXX 生成 Wrap的对象代理
		 * Wrap the given bean if necessary, i.e. if it is eligible for being proxied.
		 * @param bean the raw bean instance
		 * @param beanName the name of the bean
		 * @param cacheKey the cache key for metadata access
		 * @return a proxy wrapping the bean, or the raw bean instance as-is
		 */
		protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
			if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
				return bean;
			}
			if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
				return bean;
			}
			if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return bean;
			}
			// Create proxy if we have advice.
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
			// 是否需要进行动态代理
			if (specificInterceptors != DO_NOT_PROXY) {
				this.advisedBeans.put(cacheKey, Boolean.TRUE);
				Object proxy = createProxy(
						bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
				this.proxyTypes.put(cacheKey, proxy.getClass());
				return proxy;
			}
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}
		/**
		 * XXX 生成相关AOP代理切面参数
		 * Create an AOP proxy for the given bean.
		 * @param beanClass the class of the bean
		 * @param beanName the name of the bean
		 * @param specificInterceptors the set of interceptors that is
		 * specific to this bean (may be empty, but not null)
		 * @param targetSource the TargetSource for the proxy,
		 * already pre-configured to access the bean
		 * @return the AOP proxy for the bean
		 * @see #buildAdvisors
		 */
		protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
				@Nullable Object[] specificInterceptors, TargetSource targetSource) {
	
			if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
				AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
			}
	
			ProxyFactory proxyFactory = new ProxyFactory();
			proxyFactory.copyFrom(this);
	
			if (!proxyFactory.isProxyTargetClass()) {
				if (shouldProxyTargetClass(beanClass, beanName)) {
					proxyFactory.setProxyTargetClass(true);
				}
				else {
					evaluateProxyInterfaces(beanClass, proxyFactory);
				}
			}
			Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
			proxyFactory.addAdvisors(advisors);
			proxyFactory.setTargetSource(targetSource);
			customizeProxyFactory(proxyFactory);
			proxyFactory.setFrozen(this.freezeProxy);
			if (advisorsPreFiltered()) {
				proxyFactory.setPreFiltered(true);
			}
			return proxyFactory.getProxy(getProxyClassLoader());
	}
	
---
	

##### spring ioc 九次执行后置处理器的顺序

-   1.生命周期的九大后置处理器
第一次调用后置处理器org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation
调用的是InstantiationAwareBeanPostProcessor --> postProcessBeforeInstantiation方法
当一个bean在实例化之前，判断这个bean要不要产生一些新的对象，InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法可以返回任何类型，如果返回的对象不为null，就调用beanPostProcessor的postProcessAfterInitialization方法；后面的初始化、属性注入、实例化等方法都不会再调用
如果返回null，就正常的执行流程
在spring AOP当中，spring如果判断当前类100%不需要进行增强(判断当前bean是否是advice、pointcut、advisor、aopInfrastructureBean的子类，如果是，无需增强)，会把这个bean放到一个map中，并将value置为false，那么在后面进行增强的时候，会排除这个map中的bean


-   2.第二次调用后置处理器，该后置处理器推断使用哪个构造函数来初始化bean对象
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#determineConstructorsFromBeanPostProcessors

```
在推断使用哪一个构造函数的时候，会首先判断当前构造函数是否有@Value和@Autowired注解，如果没有，那就校验当前构造方法对应的bean和传来的beanClass是否一样，如果是同一个，就把当前构造函数赋值给defaultConstructor

在第二次调用后置处理器的时候，会返回当前可用的构造函数，由此来决定，使用哪个构造函数来创建bean
```

-   3.都三次调用后置处理器
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors
调用的是MergedBeanDefinitionPostProcessor --> postProcessMergedBeanDefinition 第三个后置处理器，这个后置处理器，待学习之后，再补充；在该后置处理器执行时，会


-   4.第四次调用后置处理器
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#getEarlyBeanReference
SmartInstantiationAwareBeanPostProcessor --> getEarlyBeanReference 第四个后置处理器(把创建的对象 放到earlySingletonObjects，解决循环依赖的)，处理循环依赖问题会用到这个后置处理器
这里通过后置处理器，暴露出一个ObjectFactory（个人理解是一个bean工厂），可以完成bean的实例化等操作

-   5.第五次调用后置处理器
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean--InstantiationAwareBeanPostProcessor--postProcessAfterInstantiation
调用的是InstantiationAwareBeanPostProcessor --> postProcessAfterInstantiation 第五个后置处理器(判断是否需要填充属性)
如果我们需要在程序中自己注入属性，可以利用这个点，在这里返回false，那么spring就不会调用下面这个后置处理器来注入属性

-   6.第六次调用后置处理器
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean--InstantiationAwareBeanPostProcessor--postProcessPropertyValues
调用的是InstantiationAwareBeanPostProcessor --> postProcessPropertyValues 第六个(处理类的属性值)

-   7.第七次调用后置处理器
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization
调用的是BeanPostProcessor --> postProcessBeforeInitialization bean初始化方法执行之前执行这个方法

-   8.第八次调用后置处理器
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization
调用的是BeanPostProcessor --> postProcessAfterInitialization bean初始化之后执行的方法(处理AOP)

-   9.第九次是在销毁bean容器的时候调用的
在调用ac.close()方法的时候，会调用该后置处理器 org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#postProcessBeforeDestruction

 

以上是spring声明周期中需要用到的几个后置处理器，后面再持续更新补充


	
[相关源码](https://github.com/kunge2013/spring-demo.git)
