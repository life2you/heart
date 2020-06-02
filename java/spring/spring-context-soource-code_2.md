#### Spring源码解读记录-2

---

##### SpringBoot准备启动1

---

```java
/**
	 * Run the Spring application, creating and refreshing a new
	 * {@link ApplicationContext}.
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return a running {@link ApplicationContext}
	 */
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

SpringApplication.run()方法根据注释大概了解是创建一个ApplicationContext并刷新,根据代码我们来一行一行解读.

```java
StopWatch stopWatch = new StopWatch();
```

StopWatch是一个简单的任务运行计时器,允许多个任务的计时。具体使用可以根据@StopWatch类的文档来使用

```java
Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
```

SpringBootExceptionReporter是一个回调接口，用于支持SpringApplication启动错误的自定义报告。报告程序通过SpringFactoriesLoader加载，并且必须使用单个ConfigurableApplicationContext参数声明一个公共构造函数

- 设置Headless模式:系统的一种配置模式,是在缺少显示屏、键盘或者鼠标时的系统配置。

- 设置系统参数[System.setProperty("java.awt.headless", "true")],SpringBoot默认配置为true.

```java
configureHeadlessProperty()
```

- 初始化SpringApplicationRunListener并创建实例赋值SpringApplicationRunListeners的成员变量listeners,SpringApplicationRunListener具体实现类为org.springframework.boot.context.event.EventPublishingRunListener,该配置为SpringBoot包中META-INF文件夹下的spring.factories中org.springframework.boot.SpringApplicationRunListener配置项.

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
-
  new SpringApplicationRunListeners(logger,
				getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
-
//启动EventPublishingRunListener
listeners.starting();
```

```java
EventPublishingRunListener:发布SpringApplication事件
在实际刷新上下文前,使用ApplicationEventMulticaster作用于实际的事件
public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
}
```

- 解析命令行参数并赋值DefaultApplicationArguments成员变量source,args.

```java
ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
```

- 根据监听器(SpringApplicationRunListener)和命令行配置参数创建并准备应用程序环境

```java
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
-
  // 创建并配置环境
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		ConfigurationPropertySources.attach(environment);
		listeners.environmentPrepared(environment);
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
					deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
```

```java
// 根据webApplicationType创建对应的服务环境实例
ConfigurableEnvironment environment = getOrCreateEnvironment();
switch (this.webApplicationType) {
		case SERVLET:
			return StandardServletEnvironment.class;
		case REACTIVE:
			return StandardReactiveWebEnvironment.class;
		default:
			return StandardEnvironment.class;
```

```java
configureEnvironment(environment, applicationArguments.getSourceArgs());
-
  // 根据addConversionService判断是否需要转换服务,
  if (this.addConversionService) {
    	// 该转换服务使用的是(懒加载)单例模式
			ConversionService conversionService = ApplicationConversionService.getSharedInstance();
			environment.setConversionService((ConfigurableConversionService) conversionService);
		}
    // 增加或更新应用环境中的默认配置及命令行参数配置
		configurePropertySources(environment, args);
    // 加载实际的运行时配置文件,该文件由配置[spring.profiles.active]指定,未配置时默认使用[application.properties/application.ymal]
		configureProfiles(environment, args);
```

- ConversionService:用于类型转换的接口,在Spring里搭配ConverterRegistry使用,有4个接口分别两两重载
  - canConvert判断是否可以转换个
  - convert将源类型转换为目标类型

```java
boolean canConvert(@Nullable Class<?> sourceType, Class<?> targetType);
boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType);
<T> T convert(@Nullable Object source, Class<T> targetType);
Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);
```

- 绑定该环境到应用

```java
bindToSpringApplication(environment);
```

-  如果有自定义环境将该环境转换成自定义环境(StandardServletEnvironment,StandardReactiveWebEnvironment,StandardEnvironment)

//CachedIntrospectionResults这个javaBeans弄的有点不是太懂了,所以暂停向后解读源码,准备先将小组件弄懂然后再拼接解读

