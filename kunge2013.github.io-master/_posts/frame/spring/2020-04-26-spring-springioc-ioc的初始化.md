---
layout: post
title:  "spring ioc的源码分析之 初始化过程"
date:   2020-04-26 00:06:05
categories: spring
tags: spring 源码
---

##### spring ioc的源码分析



###### 1.通过ClassPathXmlApplicationContext 创建Bean 对象的过程
	
-   1.创建上下文对象

		public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
					throws BeansException {
		
			super(parent);
			//初始化文件路径
			setConfigLocations(configLocations);
			if (refresh) {
			// 构建相关对象
				refresh();
			}
		}			
	
-  2.ClassPathXmlApplicationContext#setConfigLocations 初始化配置文件路径

		/**
		 * 解析文件相关路径	
		 * Set the config locations for this application context.
		 * <p>If not set, the implementation may use a default as appropriate.
		 */
		public void setConfigLocations(@Nullable String... locations) {
			if (locations != null) {
				Assert.noNullElements(locations, "Config locations must not be null");
				this.configLocations = new String[locations.length];
				for (int i = 0; i < locations.length; i++) {
					this.configLocations[i] = resolvePath(locations[i]).trim();
				}
			}
			else {
				this.configLocations = null;
			}
		}

-  3.解析并处理文件ClassPathXmlApplicationContext#refresh
	
		@Override
		public void refresh() throws BeansException, IllegalStateException {
			synchronized (this.startupShutdownMonitor) {
				// Prepare this context for refreshing.
				//准备刷新此上下文。
				prepareRefresh();
	
				// Tell the subclass to refresh the internal bean factory.
				//告诉子类刷新内部bean工厂。
				ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
	
				// Prepare the bean factory for use in this context.
				//准备bean工厂以供在此上下文中使用。
				prepareBeanFactory(beanFactory);
	
				try {
					// Allows post-processing of the bean factory in context subclasses.
					// 允许在上下文子类中对bean工厂进行后处理。
					postProcessBeanFactory(beanFactory);
	
					// Invoke factory processors registered as beans in the context.
					// 调用上下文中注册为bean的工厂处理器。
					invokeBeanFactoryPostProcessors(beanFactory);
	
					// Register bean processors that intercept bean creation.
					//注册拦截bean创建的bean处理器。
					registerBeanPostProcessors(beanFactory);
	
					// Initialize message source for this context.
					//初始化此上下文的消息源。
					initMessageSource();
	
					// Initialize event multicaster for this context.
					//为此上下文初始化事件多主机。
					initApplicationEventMulticaster();
	
					// Initialize other special beans in specific context subclasses.
					//初始化特定上下文子类中的其他特殊bean。
					onRefresh();
	
					// Check for listener beans and register them.
					//检查侦听器bean并注册它们。
					registerListeners();
	
					// Instantiate all remaining (non-lazy-init) singletons.
					//实例化所有剩余的（非延迟初始化）单例。
					finishBeanFactoryInitialization(beanFactory);
	
					// Last step: publish corresponding event.
					//最后一步：发布对应的事件。
					finishRefresh();
				}
	
				catch (BeansException ex) {
					if (logger.isWarnEnabled()) {
						logger.warn("Exception encountered during context initialization - " +
								"cancelling refresh attempt: " + ex);
					}
	
					// Destroy already created singletons to avoid dangling resources.
					//销毁已经创建的单例以避免资源悬空。
					destroyBeans();
	
					// Reset 'active' flag.
					//重置“活动”标志。
					cancelRefresh(ex);
	
					// Propagate exception to caller.
					//将异常传播到调用方。
					throw ex;
				}
	
				finally {
					// Reset common introspection caches in Spring's core, since we
					// might not ever need metadata for singleton beans anymore...
					// 重置Spring核心中的常见内省缓存，因为
					resetCommonCaches();
				}
			}
		}


###### a. 针对 org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory() 创建了beanFactory 

-  1.obtainFreshBeanFactory的源码如下, 构建 beanFactory 对象:
		
		/**
		 * 
		 * Tell the subclass to refresh the internal bean factory.
		 * @return the fresh BeanFactory instance
		 * @see #refreshBeanFactory()
		 * @see #getBeanFactory()
		 */
		protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
			refreshBeanFactory();// 刷新 或者创建beanFactory
			return getBeanFactory();
		}
		
	
-  2.默认 AbstractApplicationContext#refreshBeanFactory 销毁旧的beanFactory 创建新beanFactory

			/**
			 * This implementation performs an actual refresh of this context's underlying
			 * bean factory, shutting down the previous bean factory (if any) and
			 * initializing a fresh bean factory for the next phase of the context's lifecycle.
			 */
			@Override
			protected final void refreshBeanFactory() throws BeansException {
				if (hasBeanFactory()) {
					destroyBeans();
					closeBeanFactory();
				}
				try {
					DefaultListableBeanFactory beanFactory = createBeanFactory();
					beanFactory.setSerializationId(getId());
					customizeBeanFactory(beanFactory);
					loadBeanDefinitions(beanFactory);
					synchronized (this.beanFactoryMonitor) {
						this.beanFactory = beanFactory;
					}
				}
				catch (IOException ex) {
					throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
				}
			}	
	
-   3.AbstractRefreshableApplicationContext#createBeanFactory 默认DefaultListableBeanFactory


			/**
			 * Create an internal bean factory for this context.
			 * Called for each {@link #refresh()} attempt.
			 * <p>The default implementation creates a
			 * {@link org.springframework.beans.factory.support.DefaultListableBeanFactory}
			 * with the {@linkplain #getInternalParentBeanFactory() internal bean factory} of this
			 * context's parent as parent bean factory. Can be overridden in subclasses,
			 * for example to customize DefaultListableBeanFactory's settings.
			 * @return the bean factory for this context
			 * @see org.springframework.beans.factory.support.DefaultListableBeanFactory#setAllowBeanDefinitionOverriding
			 * @see org.springframework.beans.factory.support.DefaultListableBeanFactory#setAllowEagerClassLoading
			 * @see org.springframework.beans.factory.support.DefaultListableBeanFactory#setAllowCircularReferences
			 * @see org.springframework.beans.factory.support.DefaultListableBeanFactory#setAllowRawInjectionDespiteWrapping
			 */
			protected DefaultListableBeanFactory createBeanFactory() {
				return new DefaultListableBeanFactory(getInternalParentBeanFactory());
			}


###### 2.通过org.springframework.context.support.AbstractApplicationContext#prepareBeanFactory 配置 类加载器， 前置后置处理器(用于bean初始化的过程中的生命周期)

	/**
		 * Configure the factory's standard context characteristics,
		 * such as the context's ClassLoader and post-processors.
		 * 配置工厂的标准上下文特性，例如上下文的类加载器和后处理器。
		 * @param beanFactory the BeanFactory to configure
		 */
		protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
			// Tell the internal bean factory to use the context's class loader etc.
			// 告诉内部bean工厂使用上下文的类加载器等。
			beanFactory.setBeanClassLoader(getClassLoader());
			beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
			beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
	
			// Configure the bean factory with context callbacks.
			// 使用上下文回调配置bean工厂。
			beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
			beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
			beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
			beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
			beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
			beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
			beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
	
			// BeanFactory interface not registered as resolvable type in a plain factory.
			// MessageSource registered (and found for autowiring) as a bean.
			// BeanFactory接口未在普通工厂中注册为可解析类型。
			// MessageSource已注册（并为自动连接找到）为bean。
	
			beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
			beanFactory.registerResolvableDependency(ResourceLoader.class, this);
			beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
			beanFactory.registerResolvableDependency(ApplicationContext.class, this);
	
			// Register early post-processor for detecting inner beans as ApplicationListeners.
			// 注册早期的后处理器，以便将内部bean检测为ApplicationListeners。
			beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
	
			// Detect a LoadTimeWeaver and prepare for weaving, if found.
			// 检测LoadTimeWeaver并准备编织（如果发现）。
			if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
				beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
				// Set a temporary ClassLoader for type matching.
				beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
			}
	
			// Register default environment beans.
			//注册默认环境bean。
			if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
				beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
			}
			if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
				beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
			}
			if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
				beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
			}
		}

###### 3.通过org.springframework.context.support.AbstractApplicationContext#postProcessBeanFactory  允许在上下文子类中对bean工厂进行后处理。该方法在抽象类AbstractApplicationContext子类中可以重写，从而实现扩展

	/**
		 * Modify the application context's internal bean factory after its standard
		 * initialization. All bean definitions will have been loaded, but no beans
		 * will have been instantiated yet. This allows for registering special
		 * BeanPostProcessors etc in certain ApplicationContext implementations.
		 * @param beanFactory the bean factory used by the application context
		 * 在标准初始化之后修改应用程序上下文的内部bean工厂。所有bean定义都将被加载，但是还没有bean被实例化。这允许在特定的ApplicationContext实现中注册特殊的beanpstprocessors等。
		 */
		protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		
		}

		
###### 4.通过org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors  调用上下文中注册为bean的工厂处理器。

	/**
	 * 实例化并调用所有注册的beanfactorypostprocessorbean，如果给定显式顺序，则遵循显式顺序。必须在单例实例化之前调用。
	 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before singleton instantiation.
	 */
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}

###### 5.通过org.springframework.context.support.AbstractApplicationContext#registerBeanPostProcessors  注册拦截bean创建的bean处理器。

		/**
		 * 
		 * 实例化并注册所有beanPostProcessorbean，如果给定显式顺序，则遵循显式顺序。
		 * 必须在应用程序bean的任何实例化之前调用。
		 * Instantiate and register all BeanPostProcessor beans,
		 * respecting explicit order if given.
		 * <p>Must be called before any instantiation of application beans.
		 */
		protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
			PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
		}

###### 6.通过org.springframework.context.support.AbstractApplicationContext#initMessageSource  注册拦截bean创建的bean处理器。

	/**
	 * 初始化消息源。如果此上下文中未定义父项，则使用父项。
	 * Initialize the MessageSource.
	 * Use parent's if none defined in this context.
	 */
	protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using MessageSource [" + this.messageSource + "]");
			}
		}
		else {
			// Use empty MessageSource to be able to accept getMessage calls.
			DelegatingMessageSource dms = new DelegatingMessageSource();
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
			}
		}
	}


###### 7.通过org.springframework.context.support.AbstractApplicationContext#initApplicationEventMulticaster  为此上下文初始化事件多主机。

	/**
		 * 初始化ApplicationEventMulticaster。如果上下文中未定义，则使用SimpleApplicationEventMulticaster。
		 * Initialize the ApplicationEventMulticaster.
		 * Uses SimpleApplicationEventMulticaster if none defined in the context.
		 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
		 */
		protected void initApplicationEventMulticaster() {
			ConfigurableListableBeanFactory beanFactory = getBeanFactory();
			if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
				this.applicationEventMulticaster =
						beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
				if (logger.isTraceEnabled()) {
					logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
				}
			}
			else {
				this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
				beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
				if (logger.isTraceEnabled()) {
					logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
							"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
				}
			}
		}
		
###### 8.通过org.springframework.context.support.AbstractApplicationContext#onRefresh  初始化特定上下文子类中的其他特殊bean。		

		/**
	 * 可以重写以添加特定于上下文的刷新工作的模板方法。在初始化特殊bean时调用，然后实例化单例。
	 * Template method which can be overridden to add context-specific refresh work.
	 * Called on initialization of special beans, before instantiation of singletons.
	 * <p>This implementation is empty.
	 * @throws BeansException in case of errors
	 * @see #refresh()
	 */
	protected void onRefresh() throws BeansException {
		// For subclasses: do nothing by default.
	}
	
###### 9.通过org.springframework.context.support.AbstractApplicationContext#registerListeners  检查侦听器bean并注册它们。

		/**
	 * 添加将ApplicationListener实现为侦听器的bean。不会影响其他侦听器，这些侦听器可以添加而不是bean。
	 * Add beans that implement ApplicationListener as listeners.
	 * Doesn't affect other listeners, which can be added without being beans.
	 */
	protected void registerListeners() {
		// Register statically specified listeners first.
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
	
###### 10.通过org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization  实例化所有剩余的（非延迟初始化）

	/**
	 * 完成此上下文的bean工厂的初始化，初始化所有剩余的单例bean。
	 * Finish the initialization of this context's bean factory,
	 * initializing all remaining singleton beans.
	 */
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
	
	
###### 11.通过org.springframework.context.support.AbstractApplicationContext#finishRefresh  最后一步：发布对应的事件

		/**
	 * 完成此上下文的刷新，调用LifecycleProcessor的onRefresh（）方法并发布org.springframework.context.event.ContextRefreshedEvent。
	 * Finish the refresh of this context, invoking the LifecycleProcessor's
	 * onRefresh() method and publishing the
	 * {@link org.springframework.context.event.ContextRefreshedEvent}.
	 */
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
	
	
[相关源码](https://github.com/kunge2013/spring-demo.git)	