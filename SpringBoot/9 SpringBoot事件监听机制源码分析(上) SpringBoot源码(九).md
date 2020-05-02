**SpringBoot中文注释项目Github地址：**

https://github.com/yuanmabiji/spring-boot-2.1.0.RELEASE



本篇接 [SpringApplication对象是如何构建的？ SpringBoot源码（八）](https://juejin.im/post/5e82bac9518825737a314096)

# 1 温故而知新
温故而知新，我们来简单回顾一下上篇的内容，上一篇我们分析了**SpringApplication对象的构建过程及SpringBoot自己实现的一套SPI机制**，现将关键步骤再浓缩总结下：
1. `SpringApplication`对象的构造过程其实就是给`SpringApplication`类的**6**个成员变量赋值；
2. SpringBoot通过以下步骤实现自己的SPI机制：
* 1)首先获取线程上下文类加载器; 
* 2)然后利用上下文类加载器从`spring.factories`配置文件中**加载所有的SPI扩展实现类并放入缓存中**;
* 3)根据SPI接口从缓存中取出相应的SPI扩展实现类; 
* 4)实例化从缓存中取出的SPI扩展实现类并返回。

# 2 引言
在SpringBoot启动过程中，每个不同的启动阶段会分别广播不同的内置生命周期事件，然后相应的监听器会监听这些事件来执行一些初始化逻辑工作比如`ConfigFileApplicationListener`会监听`onApplicationEnvironmentPreparedEvent`事件来加载配置文件`application.properties`的环境变量等。

因此本篇内容将来分析下SpringBoot的事件监听机制的源码。

# 3 SpringBoot广播内置生命周期事件流程分析
为了探究SpringBoot广播内置生命周期事件流程，我们再来回顾一下SpringBoot的启动流程代码：
```java
// SpringApplication.java

public ConfigurableApplicationContext run(String... args) {
	StopWatch stopWatch = new StopWatch();
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
	configureHeadlessProperty();
	// 【0】新建一个SpringApplicationRunListeners对象用于发射SpringBoot启动过程中的生命周期事件
	SpringApplicationRunListeners listeners = getRunListeners(args);
	// 【1】》》》》》发射【ApplicationStartingEvent】事件，标志SpringApplication开始启动
	listeners.starting();
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(
				args);
		// 【2】》》》》》发射【ApplicationEnvironmentPreparedEvent】事件，此时会去加载application.properties等配置文件的环境变量，同时也有标志环境变量已经准备好的意思
		ConfigurableEnvironment environment = prepareEnvironment(listeners,
				applicationArguments);
		configureIgnoreBeanInfo(environment);
		Banner printedBanner = printBanner(environment);
		context = createApplicationContext();
		exceptionReporters = getSpringFactoriesInstances(
				SpringBootExceptionReporter.class,
				new Class[] { ConfigurableApplicationContext.class }, context); 
		// 【3】》》》》》发射【ApplicationContextInitializedEvent】事件，标志context容器被创建且已准备好
		// 【4】》》》》》发射【ApplicationPreparedEvent】事件，标志Context容器已经准备完成
		prepareContext(context, environment, listeners, applicationArguments,
				printedBanner);
		refreshContext(context);
		afterRefresh(context, applicationArguments);
		stopWatch.stop();
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass)
					.logStarted(getApplicationLog(), stopWatch);
		}
		// 【5】》》》》》发射【ApplicationStartedEvent】事件，标志spring容器已经刷新，此时所有的bean实例都已经加载完毕
		listeners.started(context);
		callRunners(context, applicationArguments);
	}
	// 【6】》》》》》发射【ApplicationFailedEvent】事件，标志SpringBoot启动失败
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}
	try {
		// 【7】》》》》》发射【ApplicationReadyEvent】事件，标志SpringApplication已经正在运行即已经成功启动，可以接收服务请求了。
		listeners.running(context);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
	return context;
}
```
可以看到SpringBoot在启动过程中首先会先新建一个`SpringApplicationRunListeners`对象用于发射SpringBoot启动过程中的各种生命周期事件，比如发射`ApplicationStartingEvent`,`ApplicationEnvironmentPreparedEvent`和`ApplicationContextInitializedEvent`等事件，然后相应的监听器会执行一些SpringBoot启动过程中的初始化逻辑。那么，监听这些SpringBoot的生命周期事件的监听器们是何时被加载实例化的呢？还记得上篇文章在分析`SpringApplication`的构建过程吗？没错，这些执行初始化逻辑的监听器们正是在`SpringApplication`的构建过程中根据`ApplicationListener`接口去`spring.factories`配置文件中加载并实例化的。
# 3.1 为广播SpringBoot内置生命周期事件做前期准备
# 3.1.1 加载ApplicationListener监听器实现类
我们再来回顾下[SpringApplication对象是如何构建的？ SpringBoot源码（八）](https://juejin.im/post/5e82bac9518825737a314096)一文中讲到在构建`SpringApplication`对象时的`setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));`这句代码。

这句代码做的事情就是从`spring.factories`中加载出`ApplicationListener`事件监听接口的SPI扩展实现类然后添加到`SpringApplication`对象的`listeners`集合中，用于后续监听SpringBoot启动过程中的事件，来执行一些初始化逻辑工作。

SpringBoot启动时的具体监听器们都实现了`ApplicationListener`接口，其在`spring.factories`部分配置如下：

![](https://user-gold-cdn.xitu.io/2020/4/12/1716f1b4e4687069?w=1150&h=498&f=png&s=75210)

不过在调试时，会从所有的spring.factories配置文件中加载监听器，最终加载了10个监听器。如下图：


![](https://user-gold-cdn.xitu.io/2020/4/13/17170eb8ae034bbd?w=712&h=584&f=png&s=53731)

# 3.1.2 加载SPI扩展类EventPublishingRunListener
前面讲到，在SpringBoot的启动过程中首先会先新建一个`SpringApplicationRunListeners`对象用于发射SpringBoot启动过程中的生命周期事件，即我们现在来看下`SpringApplicationRunListeners listeners = getRunListeners(args);`这句代码：

```java
// SpringApplication.java

private SpringApplicationRunListeners getRunListeners(String[] args) {
	// 构造一个由SpringApplication.class和String[].class组成的types
	Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
	// 1) 根据SpringApplicationRunListener接口去spring.factories配置文件中加载其SPI扩展实现类
	// 2) 构建一个SpringApplicationRunListeners对象并返回
	return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
			SpringApplicationRunListener.class, types, this, args));
}

```

我们将重点放到`getSpringFactoriesInstances(
			SpringApplicationRunListener.class, types, this, args)`这句代码，`getSpringFactoriesInstances`这个方法我们已经很熟悉，在上一篇分析SpringBoot的SPI机制时已经详细分析过这个方法。可以看到SpringBoot此时又是根据`SpringApplicationRunListener`这个SPI接口去`spring.factories`中加载相应的SPI扩展实现类，我们直接去`spring.factories`中看看`SpringApplicationRunListener`有哪些SPI实现类：
			
![](https://user-gold-cdn.xitu.io/2020/4/18/1718d054c88bebb9?w=692&h=117&f=png&s=13088)
由上图可以看到，`SpringApplicationRunListener`只有`EventPublishingRunListener`这个SPI实现类
`EventPublishingRunListener`这个哥们在SpringBoot的启动过程中尤其重要，由其在SpringBoot启动过程的不同阶段发射不同的SpringBoot的生命周期事件，**即`SpringApplicationRunListeners`对象没有承担广播事件的职责，而最终是委托`EventPublishingRunListener`这个哥们来广播事件的。**

因为从`spring.factories`中加载`EventPublishingRunListener`类后还会实例化该类，那么我们再跟进`EventPublishingRunListener`的源码，看看其是如何承担发射SpringBoot生命周期事件这一职责的？
```java
// EventPublishingRunListener.java

public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {

	private final SpringApplication application;

	private final String[] args;
	/**
	 * 拥有一个SimpleApplicationEventMulticaster事件广播器来广播事件
	 */
	private final SimpleApplicationEventMulticaster initialMulticaster;

	public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		// 新建一个事件广播器SimpleApplicationEventMulticaster对象
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		// 遍历在构造SpringApplication对象时从spring.factories配置文件中获取的事件监听器
		for (ApplicationListener<?> listener : application.getListeners()) {
			// 将从spring.factories配置文件中获取的事件监听器们添加到事件广播器initialMulticaster对象的相关集合中
			this.initialMulticaster.addApplicationListener(listener);
		}
	}

	@Override
	public int getOrder() {
		return 0;
	}
	// 》》》》》发射【ApplicationStartingEvent】事件
	@Override
	public void starting() {
		this.initialMulticaster.multicastEvent(
				new ApplicationStartingEvent(this.application, this.args));
	}
	// 》》》》》发射【ApplicationEnvironmentPreparedEvent】事件
	@Override
	public void environmentPrepared(ConfigurableEnvironment environment) {
		this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(
				this.application, this.args, environment));
	}
	// 》》》》》发射【ApplicationContextInitializedEvent】事件
	@Override
	public void contextPrepared(ConfigurableApplicationContext context) {
		this.initialMulticaster.multicastEvent(new ApplicationContextInitializedEvent(
				this.application, this.args, context));
	}
	// 》》》》》发射【ApplicationPreparedEvent】事件
	@Override
	public void contextLoaded(ConfigurableApplicationContext context) {
		for (ApplicationListener<?> listener : this.application.getListeners()) {
			if (listener instanceof ApplicationContextAware) {
				((ApplicationContextAware) listener).setApplicationContext(context);
			}
			context.addApplicationListener(listener);
		}
		this.initialMulticaster.multicastEvent(
				new ApplicationPreparedEvent(this.application, this.args, context));
	}
	// 》》》》》发射【ApplicationStartedEvent】事件
	@Override
	public void started(ConfigurableApplicationContext context) {
		context.publishEvent(
				new ApplicationStartedEvent(this.application, this.args, context));
	}
	// 》》》》》发射【ApplicationReadyEvent】事件
	@Override
	public void running(ConfigurableApplicationContext context) {
		context.publishEvent(
				new ApplicationReadyEvent(this.application, this.args, context));
	}
	// 》》》》》发射【ApplicationFailedEvent】事件
	@Override
	public void failed(ConfigurableApplicationContext context, Throwable exception) {
		ApplicationFailedEvent event = new ApplicationFailedEvent(this.application,
				this.args, context, exception);
		if (context != null && context.isActive()) {
			// Listeners have been registered to the application context so we should
			// use it at this point if we can
			context.publishEvent(event);
		}
		else {
			// An inactive context may not have a multicaster so we use our multicaster to
			// call all of the context's listeners instead
			if (context instanceof AbstractApplicationContext) {
				for (ApplicationListener<?> listener : ((AbstractApplicationContext) context)
						.getApplicationListeners()) {
					this.initialMulticaster.addApplicationListener(listener);
				}
			}
			this.initialMulticaster.setErrorHandler(new LoggingErrorHandler());
			this.initialMulticaster.multicastEvent(event);
		}
	}
	
	// ...省略非关键代码
}

```

可以看到`EventPublishingRunListener`类实现了`SpringApplicationRunListener`接口，`SpringApplicationRunListener`接口定义了SpringBoot启动时发射生命周期事件的接口方法，而`EventPublishingRunListener`类正是通过实现`SpringApplicationRunListener`接口的`starting`,`environmentPrepared`和`contextPrepared`等方法来广播SpringBoot不同的生命周期事件，我们直接看下`SpringApplicationRunListener`接口源码好了：
```java
// SpringApplicationRunListener.java

public interface SpringApplicationRunListener {

	void starting();

	void environmentPrepared(ConfigurableEnvironment environment);

	void contextPrepared(ConfigurableApplicationContext context);

	void contextLoaded(ConfigurableApplicationContext context);

	void started(ConfigurableApplicationContext context);

	void running(ConfigurableApplicationContext context);

	void failed(ConfigurableApplicationContext context, Throwable exception);
}
```
我们再接着分析`EventPublishingRunListener`这个类，可以看到其有一个重要的成员属性`initialMulticaster`，该成员属性是`SimpleApplicationEventMulticaster`类对象，该类正是承担了广播SpringBoot启动时生命周期事件的职责,**即`EventPublishingRunListener`对象没有承担广播事件的职责，而最终是委托`SimpleApplicationEventMulticaster`这个哥们来广播事件的。** 从`EventPublishingRunListener`的源码中也可以看到在`starting`,`environmentPrepared`和`contextPrepared`等方法中也正是通过调用`SimpleApplicationEventMulticaster`类对象的`multicastEvent`方法来广播事件的。

> **思考** SpringBoot启动过程中发射事件时事件广播者是层层委托职责的，起初由`SpringApplicationRunListeners`对象承担，然后`SpringApplicationRunListeners`对象将广播事件职责委托给`EventPublishingRunListener`对象，最终`EventPublishingRunListener`对象将广播事件的职责委托给`SimpleApplicationEventMulticaster`对象。**为什么要层层委托这么做呢？** 这个值得大家思考。

前面讲到从`spring.factories`中加载出`EventPublishingRunListener`类后会实例化，而实例化必然会通过`EventPublishingRunListener`的构造函数来进行实例化，因此我们接下来分析下`EventPublishingRunListener`的构造函数源码：
```java
// EventPublishingRunListener.java

public EventPublishingRunListener(SpringApplication application, String[] args) {
	this.application = application;
	this.args = args;
	// 新建一个事件广播器SimpleApplicationEventMulticaster对象
	this.initialMulticaster = new SimpleApplicationEventMulticaster();
	// 遍历在构造SpringApplication对象时从spring.factories配置文件中获取的事件监听器
	for (ApplicationListener<?> listener : application.getListeners()) {
		// 将从spring.factories配置文件中获取的事件监听器们添加到事件广播器initialMulticaster对象的相关集合中
		this.initialMulticaster.addApplicationListener(listener);
	}
}
```
可以看到在`EventPublishingRunListener`的构造函数中有一个`for`循环会遍历之前从`spring.factories`中加载的监听器们，然后添加到集合中缓存起来，用于以后广播各种事件时直接从这个集合中取出来即可，而不用再去`spring.factories`中加载，提高效率。

# 3.2 广播SpringBoot的内置生命周期事件
从`spring.factories`配置文件中加载并实例化`EventPublishingRunListener`对象后，那么在在SpringBoot的启动过程中会发射一系列SpringBoot内置的生命周期事件，我们再来回顾下SpringBoot启动过程中的源码：
```java
// SpringApplication.java

public ConfigurableApplicationContext run(String... args) {
	StopWatch stopWatch = new StopWatch();
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
	configureHeadlessProperty();
	// 【0】新建一个SpringApplicationRunListeners对象用于发射SpringBoot启动过程中的生命周期事件
	SpringApplicationRunListeners listeners = getRunListeners(args);
	// 【1】》》》》》发射【ApplicationStartingEvent】事件，标志SpringApplication开始启动
	listeners.starting();
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(
				args);
		// 【2】》》》》》发射【ApplicationEnvironmentPreparedEvent】事件，此时会去加载application.properties等配置文件的环境变量，同时也有标志环境变量已经准备好的意思
		ConfigurableEnvironment environment = prepareEnvironment(listeners,
				applicationArguments);
		configureIgnoreBeanInfo(environment);
		Banner printedBanner = printBanner(environment);
		context = createApplicationContext();
		exceptionReporters = getSpringFactoriesInstances(
				SpringBootExceptionReporter.class,
				new Class[] { ConfigurableApplicationContext.class }, context); 
		// 【3】》》》》》发射【ApplicationContextInitializedEvent】事件，标志context容器被创建且已准备好
		// 【4】》》》》》发射【ApplicationPreparedEvent】事件，标志Context容器已经准备完成
		prepareContext(context, environment, listeners, applicationArguments,
				printedBanner);
		refreshContext(context);
		afterRefresh(context, applicationArguments);
		stopWatch.stop();
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass)
					.logStarted(getApplicationLog(), stopWatch);
		}
		// 【5】》》》》》发射【ApplicationStartedEvent】事件，标志spring容器已经刷新，此时所有的bean实例都已经加载完毕
		listeners.started(context);
		callRunners(context, applicationArguments);
	}
	// 【6】》》》》》发射【ApplicationFailedEvent】事件，标志SpringBoot启动失败
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}
	try {
		// 【7】》》》》》发射【ApplicationReadyEvent】事件，标志SpringApplication已经正在运行即已经成功启动，可以接收服务请求了。
		listeners.running(context);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
	return context;
}
```
可以看到在SpringBoot的启动过程中总共会发射7种不同类型的生命周期事件，来标志SpringBoot的不同启动阶段，同时，这些生命周期事件的监听器们也会执行一些启动过程中的初始化逻辑，关于这些监听器的初始化逻辑将在下一篇内容中会分析。以下是SpringBoot启动过程中要发射的事件类型，其中`ApplicationFailedEvent`在SpringBoot启动过程中遇到异常才会发射：
1. `ApplicationStartingEvent`
2. `ApplicationEnvironmentPreparedEvent`
3. `ApplicationContextInitializedEvent`
4. `ApplicationPreparedEvent`
5. `ApplicationStartedEvent`
6. `ApplicationFailedEvent`
7. `ApplicationReadyEvent`

我们以`listeners.starting();`这句代码为例，看看`EventPublishingRunListener`对象发射事件的源码：
```java
// SpringApplicationRunListeners.java

public void starting() {
	// 遍历listeners集合，这里实质取出的就是刚才从spring.factories中取出的SPI实现类EventPublishingRunListener
	// 而EventPublishingRunListener对象承担了SpringBoot启动过程中负责广播不同的生命周期事件
	for (SpringApplicationRunListener listener : this.listeners) {
	        // 调用EventPublishingRunListener的starting方法来广播ApplicationStartingEvent事件
		listener.starting();
	}
}
```
继续跟进`listener.starting();`的源码:
```java
EventPublishingRunListener.java

// 》》》》》发射【ApplicationStartingEvent】事件
public void starting() {
	// EventPublishingRunListener对象将发布ApplicationStartingEvent这件事情委托给了initialMulticaster对象
	// 调用initialMulticaster的multicastEvent方法来发射ApplicationStartingEvent事件
	this.initialMulticaster.multicastEvent(
			new ApplicationStartingEvent(this.application, this.args));
}
```
可以看到，`EventPublishingRunListener`对象将发布`ApplicationStartingEvent`这件事情委托给了`SimpleApplicationEventMulticaster`对象`initialMulticaster`,
,而`initialMulticaster`对象最终会调用其`multicastEvent`方法来发射`ApplicationStartingEvent`事件。关于`SimpleApplicationEventMulticaster`类如何广播事件，笔者已经在[Spring是如何实现事件监听机制的？ Spring源码（二）](https://juejin.im/post/5e421bfc6fb9a07cd80f1354)这篇文章已经详细分析，这里不再赘述。

关于SpringBoot启动过程中发射其他生命周期事件的源码这里不再分析

# 4 SpringBoot的内置生命周期事件总结
好了，前面已经分析了SpringBoot启动过程中要发射的各种生命周期事件，下面列一个表格总结下：

![](https://user-gold-cdn.xitu.io/2020/4/18/1718d7dce64e30d2?w=881&h=870&f=png&s=88780)


# 5 小结
SpringBoot启动过程中广播生命周期事件的源码分析就到此结束了，下一篇会继续介绍监听这些生命周期事件的监听器们。我们再回顾本篇内容总结下关键点：

SpringBoot启动过程中会发射7种类型的生命周期事件，标志不同的启动阶段，然后相应的监听器会监听这些事件来执行一些初始化逻辑工作。

**【源码笔记】Github源码分析项目上线啦！！！下面是笔记的Github地址：**

https://github.com/yuanmabiji/Java-SourceCode-Blogs

**原创不易，帮忙Star一下呗**！

  