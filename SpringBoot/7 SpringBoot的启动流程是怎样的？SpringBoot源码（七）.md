**注：该源码分析对应SpringBoot版本为2.1.0.RELEASE**
# 1 温故而知新
本篇接 [SpringBoot内置的各种Starter是怎样构建的？  SpringBoot源码（六）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/6%20SpringBoot%E5%86%85%E7%BD%AE%E7%9A%84%E5%90%84%E7%A7%8DStarter%E6%98%AF%E6%80%8E%E6%A0%B7%E6%9E%84%E5%BB%BA%E7%9A%84%EF%BC%9F%20%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E5%85%AD%EF%BC%89.md)

温故而知新，我们来简单回顾一下上篇的内容，上一篇我们分析了**SpringBoot内置的各种Starter是怎样构建的?**，现将关键点重新回顾总结下：

1. `spring-boot-starter-xxx`起步依赖没有一行代码，而是直接或间接依赖了`xxx-autoconfigure`模块，而`xxx-autoconfigure`模块承担了`spring-boot-starter-xxx`起步依赖自动配置的实现；
2. `xxx-autoconfigure`自动配置模块引入了一些可选依赖，这些可选依赖不会被传递到`spring-boot-starter-xxx`起步依赖中，这是起步依赖构建的**关键点**；
3. `spring-boot-starter-xxx`起步依赖**显式**引入了一些对自动配置起作用的可选依赖，因此会触发 `xxx-autoconfigure`自动配置的逻辑（比如创建某些符合条件的配置`bean`）；
4. 经过前面3步的准备，我们项目只要引入了某个起步依赖后，就可以开箱即用了，而不用手动去创建一些`bean`等。

# 2 引言
本来这篇文章会继续SpringBoot自动配置的源码分析的，想分析下`spring-boot-starter-web`的自动配置的源码是怎样的的。但是考虑到`spring-boot-starter-web`的自动配置逻辑跟内置`Tomcat`等有关，因此想以后等分析了SpringBoot的内置`Tomcat`的相关源码后再来继续分析`spring-boot-starter-web`的自动配置的源码。

因此，本篇我们来探究下**SpringBoot的启动流程是怎样的？**
# 3 如何编写一个SpringBoot启动类
我们都知道，我们运行一个SpringBoot项目，引入相关`Starters`和相关依赖后，再编写一个启动类，然后在这个启动类标上`@SpringBootApplication`注解，然后就可以启动运行项目了，如下代码：
```java
//MainApplication.java

@SpringBootApplication
public class MainApplication {
	public static void main(String[] args) {
		SpringApplication.run(MainApplication.class, args);
	}
}
```
如上代码，我们在`MainApplication`启动类上标注了`@SpringBootApplication`注解，然后在`main`函数中调用`SpringApplication.run(MainApplication.class, args);`这句代码就完成了SpringBoot的启动流程，非常简单。
# 4 @SpringBootApplication
现在我们来分析下标注在启动类上的`@SpringBootApplication`注解，直接上源码：
```java
// SpringBootApplication.java 

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { // TODO 这两个排除过滤器TypeExcludeFilter和AutoConfigurationExcludeFilter暂不知道啥作用
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
        // 等同于EnableAutoConfiguration注解的exclude属性
	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};
        // 等同于EnableAutoConfiguration注解的excludeName属性
	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};
        // 等同于ComponentScan注解的basePackages属性
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};
        // 等同于ComponentScan注解的basePackageClasses属性
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};
}
```

可以看到，`@SpringBootApplication`注解是一个组合注解，主要由`@SpringBootConfiguration`,`@EnableAutoConfiguration`和`@ComponentScan`这三个注解组合而成。

因此`@SpringBootApplication`注解主要作为一个配置类，能够触发包扫描和自动配置的逻辑，从而使得SpringBoot的相关`bean`被注册进Spring容器。
# 5 SpringBoot的启动流程是怎样的？
接下来是本篇的重点，我们来分析下**SpringBoot的启动流程是怎样的？**

我们接着来看前面`main`函数里的`SpringApplication.run(MainApplication.class, args);`这句代码，那么`SpringApplication`这个类是干嘛的呢？

`SpringApplication`类是用来启动SpringBoot项目的，可以在java的`main`方法中启动，目前我们知道这些就足够了。下面看下`SpringApplication.run(MainApplication.class, args);`这句代码的源码：
```java
// SpringApplication.java

// run方法是一个静态方法，用于启动SpringBoot
public static ConfigurableApplicationContext run(Class<?> primarySource,
		String... args) {
	// 继续调用静态的run方法
	return run(new Class<?>[] { primarySource }, args);
}
```
在上面的静态`run`方法里又继续调用另一个静态`run`方法：
```java
// SpringApplication.java

// run方法是一个静态方法，用于启动SpringBoot
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
		String[] args) {
	// 构建一个SpringApplication对象，并调用其run方法来启动
	return new SpringApplication(primarySources).run(args);
}
```
如上代码，可以看到构建了一个`SpringApplication`对象，然后再调用其`run`方法来启动SpringBoot项目。关于`SpringApplication`对象是如何构建的，我们后面再分析，现在直接来看下启动流程的源码：
```java
// SpringApplication.java

public ConfigurableApplicationContext run(String... args) {
	// new 一个StopWatch用于统计run启动过程花了多少时间
	StopWatch stopWatch = new StopWatch();
	// 开始计时
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	// exceptionReporters集合用来存储异常报告器，用来报告SpringBoot启动过程的异常
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
	// 配置headless属性，即“java.awt.headless”属性，默认为ture
	// 其实是想设置该应用程序,即使没有检测到显示器,也允许其启动.对于服务器来说,是不需要显示器的,所以要这样设置.
	configureHeadlessProperty();
	// 【1】从spring.factories配置文件中加载到EventPublishingRunListener对象并赋值给SpringApplicationRunListeners
	// EventPublishingRunListener对象主要用来发射SpringBoot启动过程中内置的一些生命周期事件，标志每个不同启动阶段
	SpringApplicationRunListeners listeners = getRunListeners(args);
	// 启动SpringApplicationRunListener的监听，表示SpringApplication开始启动。
	// 》》》》》发射【ApplicationStartingEvent】事件
	listeners.starting();
	try {
		// 创建ApplicationArguments对象，封装了args参数
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(
				args);
		// 【2】准备环境变量，包括系统变量，环境变量，命令行参数，默认变量，servlet相关配置变量，随机值，
		// JNDI属性值，以及配置文件（比如application.properties）等，注意这些环境变量是有优先级的
		// 》》》》》发射【ApplicationEnvironmentPreparedEvent】事件
		ConfigurableEnvironment environment = prepareEnvironment(listeners,
				applicationArguments);
		// 配置spring.beaninfo.ignore属性，默认为true，即跳过搜索BeanInfo classes.
		configureIgnoreBeanInfo(environment);
		// 【3】控制台打印SpringBoot的bannner标志
		Banner printedBanner = printBanner(environment);
		// 【4】根据不同类型创建不同类型的spring applicationcontext容器
		// 因为这里是servlet环境，所以创建的是AnnotationConfigServletWebServerApplicationContext容器对象
		context = createApplicationContext();
		// 【5】从spring.factories配置文件中加载异常报告期实例，这里加载的是FailureAnalyzers
		// 注意FailureAnalyzers的构造器要传入ConfigurableApplicationContext，因为要从context中获取beanFactory和environment
		exceptionReporters = getSpringFactoriesInstances(
				SpringBootExceptionReporter.class,
				new Class[] { ConfigurableApplicationContext.class }, context); // ConfigurableApplicationContext是AnnotationConfigServletWebServerApplicationContext的父接口
		// 【6】为刚创建的AnnotationConfigServletWebServerApplicationContext容器对象做一些初始化工作，准备一些容器属性值等
		// 1）为AnnotationConfigServletWebServerApplicationContext的属性AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner设置environgment属性
		// 2）根据情况对ApplicationContext应用一些相关的后置处理，比如设置resourceLoader属性等
		// 3）在容器刷新前调用各个ApplicationContextInitializer的初始化方法，ApplicationContextInitializer是在构建SpringApplication对象时从spring.factories中加载的
		// 4）》》》》》发射【ApplicationContextInitializedEvent】事件，标志context容器被创建且已准备好
		// 5）从context容器中获取beanFactory，并向beanFactory中注册一些单例bean，比如applicationArguments，printedBanner
		// 6）TODO 加载bean到application context，注意这里只是加载了部分bean比如mainApplication这个bean，大部分bean应该是在AbstractApplicationContext.refresh方法中被加载？这里留个疑问先
		// 7）》》》》》发射【ApplicationPreparedEvent】事件，标志Context容器已经准备完成
		prepareContext(context, environment, listeners, applicationArguments,
				printedBanner);
		// 【7】刷新容器，这一步至关重要，以后会在分析Spring源码时详细分析，主要做了以下工作：
		// 1）在context刷新前做一些准备工作，比如初始化一些属性设置，属性合法性校验和保存容器中的一些早期事件等；
		// 2）让子类刷新其内部bean factory,注意SpringBoot和Spring启动的情况执行逻辑不一样
		// 3）对bean factory进行配置，比如配置bean factory的类加载器，后置处理器等
		// 4）完成bean factory的准备工作后，此时执行一些后置处理逻辑，子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置
		// 在这一步，所有的bean definitions将会被加载，但此时bean还不会被实例化
		// 5）执行BeanFactoryPostProcessor的方法即调用bean factory的后置处理器：
		// BeanDefinitionRegistryPostProcessor（触发时机：bean定义注册之前）和BeanFactoryPostProcessor（触发时机：bean定义注册之后bean实例化之前）
		// 6）注册bean的后置处理器BeanPostProcessor，注意不同接口类型的BeanPostProcessor；在Bean创建前后的执行时机是不一样的
		// 7）初始化国际化MessageSource相关的组件，比如消息绑定，消息解析等
		// 8）初始化事件广播器，如果bean factory没有包含事件广播器，那么new一个SimpleApplicationEventMulticaster广播器对象并注册到bean factory中
		// 9）AbstractApplicationContext定义了一个模板方法onRefresh，留给子类覆写，比如ServletWebServerApplicationContext覆写了该方法来创建内嵌的tomcat容器
		// 10）注册实现了ApplicationListener接口的监听器，之前已经有了事件广播器，此时就可以派发一些early application events
		// 11）完成容器bean factory的初始化，并初始化所有剩余的单例bean。这一步非常重要，一些bean postprocessor会在这里调用。
		// 12）完成容器的刷新工作，并且调用生命周期处理器的onRefresh()方法，并且发布ContextRefreshedEvent事件
		refreshContext(context);
		// 【8】执行刷新容器后的后置处理逻辑，注意这里为空方法
		afterRefresh(context, applicationArguments);
		// 停止stopWatch计时
		stopWatch.stop();
		// 打印日志
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass)
					.logStarted(getApplicationLog(), stopWatch);
		}
		// 》》》》》发射【ApplicationStartedEvent】事件，标志spring容器已经刷新，此时所有的bean实例都已经加载完毕
		listeners.started(context);
		// 【9】调用ApplicationRunner和CommandLineRunner的run方法，实现spring容器启动后需要做的一些东西比如加载一些业务数据等
		callRunners(context, applicationArguments);
	}
	// 【10】若启动过程中抛出异常，此时用FailureAnalyzers来报告异常
	// 并》》》》》发射【ApplicationFailedEvent】事件，标志SpringBoot启动失败
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}

	try {
		// 》》》》》发射【ApplicationReadyEvent】事件，标志SpringApplication已经正在运行即已经成功启动，可以接收服务请求了。
		listeners.running(context);
	}
	// 若出现异常，此时仅仅报告异常，而不会发射任何事件
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
	// 【11】最终返回容器
	return context;
}
```
如上代码就是SpringBoot的启动流程了，其中注释也非常详细，主要步骤也已经标注`【x】`，现将主要步骤总结如下：

1. 从`spring.factories`配置文件中**加载`EventPublishingRunListener`对象**，该对象拥有`SimpleApplicationEventMulticaster`属性，即在SpringBoot启动过程的不同阶段用来发射内置的生命周期事件;
2. **准备环境变量**，包括系统变量，环境变量，命令行参数，默认变量，`servlet`相关配置变量，随机值以及配置文件（比如`application.properties`）等;
3. 控制台**打印SpringBoot的`bannner`标志**；
4. **根据不同类型环境创建不同类型的`applicationcontext`容器**，因为这里是`servlet`环境，所以创建的是`AnnotationConfigServletWebServerApplicationContext`容器对象；
5. 从`spring.factories`配置文件中**加载`FailureAnalyzers`对象**,用来报告SpringBoot启动过程中的异常；
6. **为刚创建的容器对象做一些初始化工作**，准备一些容器属性值等，对`ApplicationContext`应用一些相关的后置处理和调用各个`ApplicationContextInitializer`的初始化方法来执行一些初始化逻辑等；
7. **刷新容器**，这一步至关重要。比如调用`bean factory`的后置处理器，注册`BeanPostProcessor`后置处理器，初始化事件广播器且广播事件，初始化剩下的单例`bean`和SpringBoot创建内嵌的`Tomcat`服务器等等重要且复杂的逻辑都在这里实现，主要步骤可见代码的注释，关于这里的逻辑会在以后的spring源码分析专题详细分析；
8. **执行刷新容器后的后置处理逻辑**，注意这里为空方法；
9. **调用`ApplicationRunner`和`CommandLineRunner`的run方法**，我们实现这两个接口可以在spring容器启动后需要的一些东西比如加载一些业务数据等;
10. **报告启动异常**，即若启动过程中抛出异常，此时用`FailureAnalyzers`来报告异常;
11. 最终**返回容器对象**，这里调用方法没有声明对象来接收。

当然在SpringBoot启动过程中，每个不同的启动阶段会分别发射不同的内置生命周期事件，比如在准备`environment`前会发射`ApplicationStartingEvent`事件，在`environment`准备好后会发射`ApplicationEnvironmentPreparedEvent`事件，在刷新容器前会发射`ApplicationPreparedEvent`事件等，总之SpringBoot总共内置了7个生命周期事件，除了标志SpringBoot的不同启动阶段外，同时一些监听器也会监听相应的生命周期事件从而执行一些启动初始化逻辑。


# 6 小结
好了，SpringBoot的启动流程就已经分析完了，这篇内容主要让我们对SpringBoot的启动流程有一个整体的认识，现在还没必要去深究每一个细节，以免丢了**主线**，现在我们对SpringBoot的启动流程有一个整体的认识即可，关于启动流程的一些重要步骤我们会在以后的源码分析中来深究。

**原创不易，帮忙Star一下呗**！

由于笔者水平有限，若文中有错误还请指出，谢谢。
