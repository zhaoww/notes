> 本文基于spring 4.3.13.RELEASE

### 关于配置
启用@Async前提需要配置@EnableAsync来启用该注解
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {
    // ...
    AdviceMode mode() default AdviceMode.PROXY;
    // ...
}
```
该注解引入了AsyncConfigurationSelector，该类继承了ImportSelector,重写selectImports方法，根据adviceMode返回相应的配置类，EnableAsync异步这边默认AdviceMode.PROXY
```java
@Override
public String[] selectImports(AdviceMode adviceMode) {
	switch (adviceMode) {
		case PROXY:
			return new String[] { ProxyAsyncConfiguration.class.getName() };
		case ASPECTJ:
			return new String[] { ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME };
		default:
			return null;
	}
}
```
进入ProxyAsyncConfiguration#asyncAdvisor方法里面进行配置了executor、exceptionHandler等信息

```java
@Bean(name = TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
	Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
	AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
	Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
	if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
		bpp.setAsyncAnnotationType(customAsyncAnnotation);
	}
	if (this.executor != null) {
		bpp.setExecutor(this.executor);
	}
	if (this.exceptionHandler != null) {
		bpp.setExceptionHandler(this.exceptionHandler);
	}
	bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
	bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
	return bpp;
}
```
可以看出具体的值并不在本类中，类继承了AbstractAsyncConfiguration

```java
/**
 * Collect any {@link AsyncConfigurer} beans through autowiring.
 * 如果实现了AsyncConfigurer接口，设置线程池的配置
 */
@Autowired(required = false)
void setConfigurers(Collection<AsyncConfigurer> configurers) {
	if (CollectionUtils.isEmpty(configurers)) {
		return;
	}
	if (configurers.size() > 1) {
		throw new IllegalStateException("Only one AsyncConfigurer may exist");
	}
	AsyncConfigurer configurer = configurers.iterator().next();
	this.executor = configurer.getAsyncExecutor();
	this.exceptionHandler = configurer.getAsyncUncaughtExceptionHandler();
}
```
配置传入的是AsyncConfigurer集合，所以想要自定义线程池需要实现AsyncConfigurer接口，示例见EnableAsync注释内容

```java
/* <pre class="code">
 * &#064;Configuration
 * &#064;EnableAsync
 * public class AppConfig implements AsyncConfigurer {
 *
 *     &#064;Override
 *     public Executor getAsyncExecutor() {
 *         ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
 *         executor.setCorePoolSize(7);
 *         executor.setMaxPoolSize(42);
 *         executor.setQueueCapacity(11);
 *         executor.setThreadNamePrefix("MyExecutor-");
 *         executor.initialize();
 *         return executor;
 *     }
 *
 *     &#064;Override
 *     public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
 *         return MyAsyncUncaughtExceptionHandler();
 *     }
 * }</pre>
```
### 关于执行
spring刷新上下文最重要的方法是AbstractApplicationContext#refresh
```java
@Override
public void refresh() throws BeansException, IllegalStateException {
	// ...
	// Invoke factory processors registered as beans in the context.
	invokeBeanFactoryPostProcessors(beanFactory);
	// Register bean processors that intercept bean creation.
	registerBeanPostProcessors(beanFactory);
	// ...
}
```
对外暴露了两种后置处理器，BeanFactoryPostProcessor和BeanPostProcessor。异步这边使用的后置处理器是AsyncAnnotationBeanPostProcessor
AsyncAnnotationBeanPostProcessor处理器父类AbstractAdvisingBeanPostProcessor#postProcessAfterInitialization方法进行对bean进行后置处理，生成代理对象
```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
	if (bean instanceof AopInfrastructureBean) {
		// Ignore AOP infrastructure such as scoped proxies.
		return bean;
	}

	if (bean instanceof Advised) {
		Advised advised = (Advised) bean;
		if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
			// Add our local Advisor to the existing proxy's Advisor chain...
			if (this.beforeExistingAdvisors) {
				advised.addAdvisor(0, this.advisor);
			}
			else {
				advised.addAdvisor(this.advisor);
			}
			return bean;
		}
	}

	if (isEligible(bean, beanName)) {
		ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
		if (!proxyFactory.isProxyTargetClass()) {
			evaluateProxyInterfaces(bean.getClass(), proxyFactory);
		}
		proxyFactory.addAdvisor(this.advisor);
		customizeProxyFactory(proxyFactory);
		return proxyFactory.getProxy(getProxyClassLoader());
	}

	// No async proxy needed.
	return bean;
}
```
核心执行方法AsyncExecutionInterceptor#invoke
```
@Override
public Object invoke(final MethodInvocation invocation) throws Throwable {
	Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
	Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
	final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);

	AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
	if (executor == null) {
		throw new IllegalStateException(
				"No executor specified and no default executor set on AsyncExecutionInterceptor either");
	}

	Callable<Object> task = new Callable<Object>() {
		@Override
		public Object call() throws Exception {
			try {
				Object result = invocation.proceed();
				if (result instanceof Future) {
					return ((Future<?>) result).get();
				}
			}
			catch (ExecutionException ex) {
				handleError(ex.getCause(), userDeclaredMethod, invocation.getArguments());
			}
			catch (Throwable ex) {
				handleError(ex, userDeclaredMethod, invocation.getArguments());
			}
			return null;
		}
	};
    // 提交任务
	return doSubmit(task, executor, invocation.getMethod().getReturnType());
}

/**
 * 获取异步线程池，如果没有配置那么返回默认线程池
 */
protected AsyncTaskExecutor determineAsyncExecutor(Method method) {
    // this.executors是方法和线程池的键值对 ConcurrentHashMap<Method, AsyncTaskExecutor>
	AsyncTaskExecutor executor = this.executors.get(method);
	if (executor == null) {
		Executor targetExecutor;
		// @Async注解的value属性
		String qualifier = getExecutorQualifier(method);
		// 设置了value那么直接找该线程池
		if (StringUtils.hasLength(qualifier)) {
			targetExecutor = findQualifiedExecutor(this.beanFactory, qualifier);
		}
		else {
			targetExecutor = this.defaultExecutor;
			if (targetExecutor == null) {
				synchronized (this.executors) {
					if (this.defaultExecutor == null) {
						this.defaultExecutor = getDefaultExecutor(this.beanFactory);
					}
					targetExecutor = this.defaultExecutor;
				}
			}
		}
		if (targetExecutor == null) {
			return null;
		}
		executor = (targetExecutor instanceof AsyncListenableTaskExecutor ?
				(AsyncListenableTaskExecutor) targetExecutor : new TaskExecutorAdapter(targetExecutor));
		this.executors.put(method, executor);
	}
	return executor;
}

/**
 * 返回线程池名称， @Async注解的value属性
 */
@Override
protected String getExecutorQualifier(Method method) {
	// Maintainer's note: changes made here should also be made in
	// AnnotationAsyncExecutionAspect#getExecutorQualifier
	Async async = AnnotatedElementUtils.findMergedAnnotation(method, Async.class);
	if (async == null) {
		async = AnnotatedElementUtils.findMergedAnnotation(method.getDeclaringClass(), Async.class);
	}
	return (async != null ? async.value() : null);
}


/**
 * 获取默认线程池 
 */
protected Executor getDefaultExecutor(BeanFactory beanFactory) {
	if (beanFactory != null) {
	    // 先根据类型查找
		try {
			// Search for TaskExecutor bean... not plain Executor since that would
			// match with ScheduledExecutorService as well, which is unusable for
			// our purposes here. TaskExecutor is more clearly designed for it.
			return beanFactory.getBean(TaskExecutor.class);
		}
		catch (NoUniqueBeanDefinitionException ex) {
		    // 多个根据名称获取
			logger.debug("Could not find unique TaskExecutor bean", ex);
			try {
			    // 默认bean taskExecutor
				return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
			}
			catch (NoSuchBeanDefinitionException ex2) {
				if (logger.isInfoEnabled()) {
					logger.info("More than one TaskExecutor bean found within the context, and none is named " +
							"'taskExecutor'. Mark one of them as primary or name it 'taskExecutor' (possibly " +
							"as an alias) in order to use it for async processing: " + ex.getBeanNamesFound());
				}
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
		    // 没有找到 取默认线程池
			logger.debug("Could not find default TaskExecutor bean", ex);
			try {
			    // 默认bean taskExecutor
				return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
			}
			catch (NoSuchBeanDefinitionException ex2) {
				logger.info("No task executor bean found for async processing: " +
						"no bean of type TaskExecutor and no bean named 'taskExecutor' either");
			}
			// Giving up -> either using local default executor or none at all...
		}
	}
	return null;
}

/**
 * 返回值是Future/ListenableFuture/null，所以@Async的方法返回类型只能是Future/ListenableFuture/void
 */
protected Object doSubmit(Callable<Object> task, AsyncTaskExecutor executor, Class<?> returnType) {
	if (completableFuturePresent) {
		Future<Object> result = CompletableFutureDelegate.processCompletableFuture(returnType, task, executor);
		if (result != null) {
			return result;
		}
	}
	if (ListenableFuture.class.isAssignableFrom(returnType)) {
		return ((AsyncListenableTaskExecutor) executor).submitListenable(task);
	}
	else if (Future.class.isAssignableFrom(returnType)) {
		return executor.submit(task);
	}
	else {
		executor.submit(task);
		return null;
	}
}
```
【完】