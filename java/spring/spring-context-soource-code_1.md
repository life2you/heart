#### Spring源码解读记录-1

---

#### SpringApplication实例创建

---

##### 该系列记录从SpringBoot v:2.2.6.RELEASE源码读取记录

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
	setInitializers((Collection)getSpringFactoriesInstances(ApplicationContextInitializer.class));
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

以上方法为SpringApplication实例的创建,及部分属性赋值

```java
//判断应用以什么方式启动
this.webApplicationType = WebApplicationType.deduceFromClasspath();
WebApplicationType:
					NONE //不以Web，嵌入式Web方式启动
          SERVLET //基于Servlet Web方式启动
					REACTIVE //基于反应式Web方式启动
```

```java
//设置应用上下文初始化器
setInitializers((Collection)getSpringFactoriesInstances(ApplicationContextInitializer.class));
ApplicationContextInitializer:
// 在本次赋值初中会获取到7个SpringBoot默认的初始化器
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer
org.springframework.boot.context.ContextIdApplicationContextInitializer
org.springframework.boot.context.config.DelegatingApplicationContextInitializer
org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
  
// 以上初始化器是SpringBoot,SpringBootAutoConfigure包中META-INF文件夹下的spring.factories中org.springframework.context.ApplicationContextInitializer的配置项

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
}  
// 该方法为获取到 spring.factories中配置的初始化器实现类名称
SpringFactoriesLoader.loadFactoryNames(type, classLoader)
// 将获取的初始化器根据类名称进行实例创建
createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names)
```

**默认初始化器的作用**:

| 类名                                               | 作用                                                         |
| -------------------------------------------------- | ------------------------------------------------------------ |
| ConfigurationWarningsApplicationContextInitializer | 报告常见配置错误的警告                                       |
| ContextIdApplicationContextInitializer             | 设置应用程序的ID默认使用spring.application.name配置，若未配置则设置为:[application] |
| DelegatingApplicationContextInitializer            | 获取context.initializer.classes配置,进行自定初始化器实例创建,未配置则不做操作 |
| RSocketPortInfoApplicationContextInitializer       | 获取local.rsocket.server.port配置进行[RSocket](https://juejin.im/post/5d9b50595188255aa15d37a4)端口配置 |
| ServerPortInfoApplicationContextInitializer        | 将内置服务容器实际监听端口写入Environment环境在,该端口属性是可以使用@Value("${local.server.port}")在应用使用中可以获取到 |
| SharedMetadataReaderFactoryContextInitializer      | 创建一个SpringBoot和ConfigurationClassPostProcessor共用的CachingMetadataReaderFactory对象。实现类为：ConcurrentReferenceCachingMetadataReaderFactory |
| ConditionEvaluationReportLoggingListener           | 当Application崩溃时,会进行DEBUG日志警告                      |

```java
//设置应用监听器
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
ApplicationListener:
//在本次容器初始化时会加载11个默认监听器:
org.springframework.boot.ClearCachesApplicationListener
org.springframework.boot.builder.ParentContextCloserApplicationListener
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor
org.springframework.boot.context.FileEncodingApplicationListener
org.springframework.boot.context.config.AnsiOutputApplicationListener
org.springframework.boot.context.config.ConfigFileApplicationListener
org.springframework.boot.context.config.DelegatingApplicationListener
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener
org.springframework.boot.context.logging.LoggingApplicationListener
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
org.springframework.boot.autoconfigure.BackgroundPreinitializer
//以上是SpringBoot,SpringBootAutoConfigure包中META-INF文件夹下的spring.factories中org.springframework.context.ApplicationListener的配置项
```

**默认监听器的作用**:

| 类名                                       | 作用                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| ClearCachesApplicationListener             | 加载应用上下文后清除缓存的监听器                             |
| ParentContextCloserApplicationListener     | 如果其父级已关闭，则关闭当前应用程序上下文的监听器[使用弱引用进行判断] |
| CloudFoundryVcapEnvironmentPostProcessor   |                                                              |
| FileEncodingApplicationListener            | 如果系统文件编码[System.getProperty("file.encoding")]与环境预初设置编码[environment.getProperty("spring.mandatory-file-encoding")]不同时,将禁止启动应用 |
| AnsiOutputApplicationListener              | 设置控制台彩色输出                                           |
| ConfigFileApplicationListener              | 根据配置文件加载Environment和SpringContext,读取配置文件为application.properties/application.ymlconfig文件目录优先顺序为:               1:file:./config/,2:file:./,3:classpath:config/,4:classpath: |
| DelegatingApplicationListener              | 加载自定义监听器(读取配置参数:context.listener.classes)并转发该事件 |
| ClasspathLoggingApplicationListener        | 对环境准备时间及环境加载失败事件进行ClassPath路径打印[DEBUG] |
| LoggingApplicationListener                 | 在合适的事件触发时:1:应用程序启动时,2:应用环境准备时,3:应用准备时,4:上线文关闭时,5:应用程序启动失败时初始化,装置或卸载日志系统 |
| LiquibaseServiceLocatorApplicationListener | 如果classpath中存在类[liquibase.servicelocator.CustomResolverServiceLocator],那么将其替换成一个适用于SpringBoot的版本.[ServiceLocator(服务定位模式)](https://www.cnblogs.com/gaochundong/archive/2013/04/12/service_locator_pattern.html) |
| BackgroundPreinitializer                   | 将耗时较久的任务提前使用后台线程提前进行初始化,根据系统配置属性[spring.backgroundpreinitializer.ignore]为true时则忽略。 |

以上是创建SpringApplication实例时并将成员变量赋值的代码解读.