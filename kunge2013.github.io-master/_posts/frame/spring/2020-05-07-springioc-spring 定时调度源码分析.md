---
layout: post
title:  "spring 定时任务调度源码分析"
date:   2020-05-07 00:06:05
categories: spring
tags: spring 源码
---

##### spring 定时调度配置

-   a.启用定时调度 @EnableScheduling


-   b.配置corn表达式 @Scheduled(cron = "*/10 * * * * *")


-   c.示例如下

		@Configuration
		@EnableScheduling
		public class ScheduleService {
	
		long prefix = System.currentTimeMillis();
		ScheduledMethodRunnable r;
		@Scheduled(cron = "*/10 * * * * *")
		public void doTask() {
			long now = System.currentTimeMillis();
			Logger.getGlobal().setLevel(Level.INFO);
			Logger.getGlobal().info("测试日志输出");
			Logger logger = Logger.getLogger("JUL");
			logger.setLevel(Level.INFO);
			logger.info("dotask ....." + (prefix - (now)) / 1000);
			prefix = now;
	
		}
	}


---


##### 由于定时调度任务是通过第八次调用后置处理器，直接切入到第八次后置处理器源码进行分析

-   1.首先当bean 初始化之后，在第八次调用后置处理器时候会进行完成 定时任务的装配

		/**
		 * XXX 第八次调用后置处理器 生成动态代理bean对象
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
				Object current = processor.postProcessAfterInitialization(result, beanName);
				if (current == null) {
					return result;
				}
				result = current;
			}
			return result;
		}

-   2.装配过程 org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor#postProcessAfterInitialization

		/**
			 * @describle 查看当前bean 是否用到了 Scheduled 的相关注解，如果用到了就直接创建新的 task ，并根据表达式生成相关的执行时间
			 */
			@Override
			public Object postProcessAfterInitialization(Object bean, String beanName) {
				if (bean instanceof AopInfrastructureBean || bean instanceof TaskScheduler ||
						bean instanceof ScheduledExecutorService) {
					// Ignore AOP infrastructure such as scoped proxies.
					return bean;
				}
			Class<?> targetClass = AopProxyUtils.ultimateTargetClass(bean);
			if (!this.nonAnnotatedClasses.contains(targetClass) &&
					AnnotationUtils.isCandidateClass(targetClass, Arrays.asList(Scheduled.class, Schedules.class))) {
				/**
				 * @describle	获取 当前bean 的方法中所有的Scheduled注解
				 */
				Map<Method, Set<Scheduled>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
						(MethodIntrospector.MetadataLookup<Set<Scheduled>>) method -> {
							Set<Scheduled> scheduledMethods = AnnotatedElementUtils.getMergedRepeatableAnnotations(
									method, Scheduled.class, Schedules.class);
							return (!scheduledMethods.isEmpty() ? scheduledMethods : null);
						});
				if (annotatedMethods.isEmpty()) {
					this.nonAnnotatedClasses.add(targetClass);
					if (logger.isTraceEnabled()) {
						logger.trace("No @Scheduled annotations found on bean class: " + targetClass);
					}
				}
				else {
					/**
					 * @describle 解析当前方法转化为 task对象，并放到线程池中 
					 */
					// Non-empty set of methods
					annotatedMethods.forEach((method, scheduledMethods) ->
							scheduledMethods.forEach(scheduled -> processScheduled(scheduled, method, bean)));
					if (logger.isTraceEnabled()) {
						logger.trace(annotatedMethods.size() + " @Scheduled methods processed on bean '" + beanName +
								"': " + annotatedMethods);
					}
				}
			}
			return bean;
		}



-   3.具体装配流程如下代码 org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor#processScheduled


		/**
			 * 
			 * @describle 装配定时任务
			 * Process the given {@code @Scheduled} method declaration on the given bean.
			 * @param scheduled the @Scheduled annotation
			 * @param method the method that the annotation has been declared on
			 * @param bean the target bean instance
			 * @see #createRunnable(Object, Method)
			 */
			protected void processScheduled(Scheduled scheduled, Method method, Object bean) {
				try {
					Runnable runnable = createRunnable(bean, method);
					boolean processedSchedule = false;
					String errorMessage =
							"Exactly one of the 'cron', 'fixedDelay(String)', or 'fixedRate(String)' attributes is required";
	
				Set<ScheduledTask> tasks = new LinkedHashSet<>(4);
	
				// Determine initial delay
				long initialDelay = scheduled.initialDelay();
				String initialDelayString = scheduled.initialDelayString();
				if (StringUtils.hasText(initialDelayString)) {
					Assert.isTrue(initialDelay < 0, "Specify 'initialDelay' or 'initialDelayString', not both");
					if (this.embeddedValueResolver != null) {
						initialDelayString = this.embeddedValueResolver.resolveStringValue(initialDelayString);
					}
					if (StringUtils.hasLength(initialDelayString)) {
						try {
							initialDelay = parseDelayAsLong(initialDelayString);
						}
						catch (RuntimeException ex) {
							throw new IllegalArgumentException(
									"Invalid initialDelayString value \"" + initialDelayString + "\" - cannot parse into long");
						}
					}
				}
	
				// Check cron expression
				String cron = scheduled.cron();
				if (StringUtils.hasText(cron)) {
					String zone = scheduled.zone();
					if (this.embeddedValueResolver != null) {
						cron = this.embeddedValueResolver.resolveStringValue(cron);
						zone = this.embeddedValueResolver.resolveStringValue(zone);
					}
					if (StringUtils.hasLength(cron)) {
						Assert.isTrue(initialDelay == -1, "'initialDelay' not supported for cron triggers");
						processedSchedule = true;
						if (!Scheduled.CRON_DISABLED.equals(cron)) {
							TimeZone timeZone;
							if (StringUtils.hasText(zone)) {
								timeZone = StringUtils.parseTimeZoneString(zone);
							}
							else {
								timeZone = TimeZone.getDefault();
							}
							/**
							 * @describle 将当前的任务丢到定时线程池中，按照时间进行执行当前的任务
							 */
							tasks.add(this.registrar.scheduleCronTask(new CronTask(runnable, new CronTrigger(cron, timeZone))));
						}
					}
				}
	
				// At this point we don't need to differentiate between initial delay set or not anymore
				if (initialDelay < 0) {
					initialDelay = 0;
				}
	
				// Check fixed delay
				long fixedDelay = scheduled.fixedDelay();
				if (fixedDelay >= 0) {
					Assert.isTrue(!processedSchedule, errorMessage);
					processedSchedule = true;
					tasks.add(this.registrar.scheduleFixedDelayTask(new FixedDelayTask(runnable, fixedDelay, initialDelay)));
				}
				String fixedDelayString = scheduled.fixedDelayString();
				if (StringUtils.hasText(fixedDelayString)) {
					if (this.embeddedValueResolver != null) {
						fixedDelayString = this.embeddedValueResolver.resolveStringValue(fixedDelayString);
					}
					if (StringUtils.hasLength(fixedDelayString)) {
						Assert.isTrue(!processedSchedule, errorMessage);
						processedSchedule = true;
						try {
							fixedDelay = parseDelayAsLong(fixedDelayString);
						}
						catch (RuntimeException ex) {
							throw new IllegalArgumentException(
									"Invalid fixedDelayString value \"" + fixedDelayString + "\" - cannot parse into long");
						}
						tasks.add(this.registrar.scheduleFixedDelayTask(new FixedDelayTask(runnable, fixedDelay, initialDelay)));
					}
				}
	
				// Check fixed rate
				long fixedRate = scheduled.fixedRate();
				if (fixedRate >= 0) {
					Assert.isTrue(!processedSchedule, errorMessage);
					processedSchedule = true;
					tasks.add(this.registrar.scheduleFixedRateTask(new FixedRateTask(runnable, fixedRate, initialDelay)));
				}
				String fixedRateString = scheduled.fixedRateString();
				if (StringUtils.hasText(fixedRateString)) {
					if (this.embeddedValueResolver != null) {
						fixedRateString = this.embeddedValueResolver.resolveStringValue(fixedRateString);
					}
					if (StringUtils.hasLength(fixedRateString)) {
						Assert.isTrue(!processedSchedule, errorMessage);
						processedSchedule = true;
						try {
							fixedRate = parseDelayAsLong(fixedRateString);
						}
						catch (RuntimeException ex) {
							throw new IllegalArgumentException(
									"Invalid fixedRateString value \"" + fixedRateString + "\" - cannot parse into long");
						}
						tasks.add(this.registrar.scheduleFixedRateTask(new FixedRateTask(runnable, fixedRate, initialDelay)));
					}
				}
	
				// Check whether we had any attribute set
				Assert.isTrue(processedSchedule, errorMessage);
	
				// Finally register the scheduled tasks
				synchronized (this.scheduledTasks) {
					Set<ScheduledTask> regTasks = this.scheduledTasks.computeIfAbsent(bean, key -> new LinkedHashSet<>(4));
					regTasks.addAll(tasks);
				}
			}
			catch (IllegalArgumentException ex) {
				throw new IllegalStateException(
						"Encountered invalid @Scheduled method '" + method.getName() + "': " + ex.getMessage());
			}
		}


-   4.针对org.springframework.scheduling.config.ScheduledTaskRegistrar#afterPropertiesSet() 第三步的过程 中的， registrar( ScheduledTaskRegistrar) 
	对象实现了InitializingBean的方法，当所有的bean 初始化完毕后会执行一次 afterPropertiesSet  依次调用 scheduleTasks()方法创建定时任务线程池，并初始化任务调度的过程

		@Override
		public void afterPropertiesSet() {
			scheduleTasks();
		}
		/**
			 * Schedule all registered tasks against the underlying
			 * {@linkplain #setTaskScheduler(TaskScheduler) task scheduler}.
			 *
			 * @Describle 任务调度线程池初始化,添加任务到线程池中
			 * 
			 */
			@SuppressWarnings("deprecation")
			protected void scheduleTasks() {
				if (this.taskScheduler == null) {
					this.localExecutor = Executors.newSingleThreadScheduledExecutor();
					this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
				}
				if (this.triggerTasks != null) {
					for (TriggerTask task : this.triggerTasks) {
						addScheduledTask(scheduleTriggerTask(task));
					}
				}
				if (this.cronTasks != null) {
					for (CronTask task : this.cronTasks) {
						addScheduledTask(scheduleCronTask(task));
					}
				}
				if (this.fixedRateTasks != null) {
					for (IntervalTask task : this.fixedRateTasks) {
						addScheduledTask(scheduleFixedRateTask(task));
					}
				}
				if (this.fixedDelayTasks != null) {
					for (IntervalTask task : this.fixedDelayTasks) {
						addScheduledTask(scheduleFixedDelayTask(task));
					}
				}
			}


-   5.添加任务到线程池中org.springframework.scheduling.config#scheduleTriggerTask()，代码如下


		/**
			 * Schedule the specified trigger task, either right away if possible
			 * or on initialization of the scheduler.
			 * @return a handle to the scheduled task, allowing to cancel it
			 * @since 4.3
			 */
			@Nullable
			public ScheduledTask scheduleTriggerTask(TriggerTask task) {
				ScheduledTask scheduledTask = this.unresolvedTasks.remove(task);
				boolean newTask = false;
				if (scheduledTask == null) {
					scheduledTask = new ScheduledTask(task);
					newTask = true;
				}
				/**
				 *@describle 添加任务到线程池
				 */
				if (this.taskScheduler != null) {
					scheduledTask.future = this.taskScheduler.schedule(task.getRunnable(), task.getTrigger());
				}
				else {
					addTriggerTask(task);
					this.unresolvedTasks.put(task, scheduledTask);
				}
				return (newTask ? scheduledTask : null);
			}


-   6.由于taskScheduler是一个org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler对象,再第5步的时候 会调用如下方法


		@Override
		@Nullable
		public ScheduledFuture<?> schedule(Runnable task, Trigger trigger) {
			ScheduledExecutorService executor = getScheduledExecutor();
			try {
				ErrorHandler errorHandler = this.errorHandler;
				if (errorHandler == null) {
					errorHandler = TaskUtils.getDefaultErrorHandler(true);
				}
				return new ReschedulingRunnable(task, trigger, executor, errorHandler).schedule();
			}
			catch (RejectedExecutionException ex) {
				throw new TaskRejectedException("Executor [" + executor + "] did not accept task: " + task, ex);
			}
		}


-   7.当新建了任务ReschedulingRunnable后任务会循环对象中调用，因此就实现了定时任务的功能，具体核心代码如下，
 this.executor.schedule(this, initialDelay, TimeUnit.MILLISECONDS)->run() {schedule();}


		/**
			 * @descriple 将任务丢到定时执行任务的线程池中，计算下次执行任务的时间, 将任务丢到线程池中,此处一直通过run方法循环调用，达到把任务丢进定时线程池的效果
			 * 
			 * @return
			 */
			@Nullable
			public ScheduledFuture<?> schedule() {
				synchronized (this.triggerContextMonitor) {
					this.scheduledExecutionTime = this.trigger.nextExecutionTime(this.triggerContext);
					if (this.scheduledExecutionTime == null) {
						return null;
					}
					long initialDelay = this.scheduledExecutionTime.getTime() - System.currentTimeMillis();
					// 将任务丢到线程池中,此处一直通过run方法循环调用，达到把任务丢进定时线程池的效果
					this.currentFuture = this.executor.schedule(this, initialDelay, TimeUnit.MILLISECONDS);
					return this;
				}
			}
		/**
			 * @descriple 执行当前本次任务 ， 添加下次执行的时间任务 
			 */
			@Override
			public void run() {
				Date actualExecutionTime = new Date();
				super.run();
				Date completionTime = new Date();
				synchronized (this.triggerContextMonitor) {
					Assert.state(this.scheduledExecutionTime != null, "No scheduled execution");
					this.triggerContext.update(this.scheduledExecutionTime, actualExecutionTime, completionTime);
					if (!obtainCurrentFuture().isCancelled()) {
						//TODO  添加下次执行的时间任务 
						schedule();
					}
				}
			}
		
[相关源码](https://github.com/kunge2013/spring-demo.git)
