**注：该源码分析对应SpringBoot版本为2.1.0.RELEASE**

本篇接 [SpringBoot的启动流程是怎样的？SpringBoot源码（七）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/7%20SpringBoot%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84%EF%BC%9FSpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E4%B8%83%EF%BC%89.md)

# 1 温故而知新

温故而知新，我们来简单回顾一下上篇的内容，上一篇我们分析了**SpringBoot的启动流程**，现将关键步骤再浓缩总结下：

1. 构建`SpringApplication`对象，用于启动SpringBoot；
2. 从`spring.factories`配置文件中加载`EventPublishingRunListener`对象用于在不同的启动阶段发射不同的生命周期事件；
3. 准备环境变量，包括系统变量，环境变量，命令行参数及配置文件（比如`application.properties`）等；
4. 创建容器`ApplicationContext`;
5. 为第4步创建的容器对象做一些初始化工作，准备一些容器属性值等，同时调用各个`ApplicationContextInitializer`的初始化方法来执行一些初始化逻辑等；
6. 刷新容器，这一步至关重要，是重点中的重点，太多复杂逻辑在这里实现；
7. 调用`ApplicationRunner`和`CommandLineRunner`的run方法，可以实现这两个接口在容器启动后来加载一些业务数据等;

在SpringBoot启动过程中，每个不同的启动阶段会分别发射不同的内置生命周期事件，然后相应的监听器会监听这些事件来执行一些初始化逻辑工作比如`ConfigFileApplicationListener`会监听`onApplicationEnvironmentPreparedEvent`事件来加载环境变量等。

# 2 引言

上篇文章在讲解SpringBoot的启动流程中，我们有看到新建了一个`SpringApplication`对象用来启动SpringBoot项目。那么，我们今天就来看看`SpringApplication`对象的构建过程，同时讲解一下SpringBoot自己实现的SPI机制。

# 3 SpringApplication对象的构建过程

本小节开始讲解`SpringApplication`对象的构造过程，因为一个对象的构造无非就是在其构造函数里给它的一些成员属性赋值，很少包含其他额外的业务逻辑（当然有时候我们可能也会在构造函数里开启一些线程啥的）。那么，我们先来看下构造`SpringApplication`对象时需要用到的一些成员属性哈：

```java
// SpringApplication.java

/**
 * SpringBoot的启动类即包含main函数的主类
 */
private Set<Class<?>> primarySources;
/**
 * 包含main函数的主类
 */
private Class<?> mainApplicationClass;
/**
 * 资源加载器
 */
private ResourceLoader resourceLoader;
/**
 * 应用类型
 */
private WebApplicationType webApplicationType;
/**
 * 初始化器
 */
private List<ApplicationContextInitializer<?>> initializers;
/**
 * 监听器
 */
private List<ApplicationListener<?>> listeners;
```

可以看到构建`SpringApplication`对象时主要是给上面代码中的六个成员属性赋值，现在我接着来看`SpringApplication`对象的构造过程。

我们先回到上一篇文章讲解的构建`SpringApplication`对象的代码处:

```java
// SpringApplication.java

// run方法是一个静态方法，用于启动SpringBoot
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
		String[] args) {
	// 构建一个SpringApplication对象，并调用其run方法来启动
	return new SpringApplication(primarySources).run(args);
}
```

跟进`SpringApplication`的构造函数中：

```java
// SpringApplication.java

public SpringApplication(Class<?>... primarySources) {
    // 继续调用SpringApplication另一个构造函数
	this(null, primarySources);
}
```

继续跟进`SpringApplication`另一个构造函数：

```java
// SpringApplication.java

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
	// 【1】给resourceLoader属性赋值，注意传入的resourceLoader参数为null
	this.resourceLoader = resourceLoader;
	Assert.notNull(primarySources, "PrimarySources must not be null");
	// 【2】给primarySources属性赋值，传入的primarySources其实就是SpringApplication.run(MainApplication.class, args);中的MainApplication.class
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
	// 【3】给webApplicationType属性赋值，根据classpath中存在哪种类型的类来确定是哪种应用类型
	this.webApplicationType = WebApplicationType.deduceFromClasspath();
	// 【4】给initializers属性赋值，利用SpringBoot自定义的SPI从spring.factories中加载ApplicationContextInitializer接口的实现类并赋值给initializers属性
	setInitializers((Collection) getSpringFactoriesInstances(
			ApplicationContextInitializer.class));
	// 【5】给listeners属性赋值，利用SpringBoot自定义的SPI从spring.factories中加载ApplicationListener接口的实现类并赋值给listeners属性
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	// 【6】给mainApplicationClass属性赋值，即这里要推断哪个类调用了main函数，然后再赋值给mainApplicationClass属性，用于后面启动流程中打印一些日志。
	this.mainApplicationClass = deduceMainApplicationClass();
}
```

可以看到构建`SpringApplication`对象时其实就是给前面讲的6个`SpringApplication`类的成员属性赋值而已，做一些初始化工作：

1. **给`resourceLoader`属性赋值**，`resourceLoader`属性，资源加载器，此时传入的`resourceLoader`参数为`null`；
2. **给`primarySources`属性赋值**，`primarySources`属性`即SpringApplication.run(MainApplication.class,args);`中传入的`MainApplication.class`，该类为SpringBoot项目的启动类，主要通过该类来扫描`Configuration`类加载`bean`；
3. **给`webApplicationType`属性赋值**，`webApplicationType`属性，代表应用类型，根据`classpath`存在的相应`Application`类来判断。因为后面要根据`webApplicationType`来确定创建哪种`Environment`对象和创建哪种`ApplicationContext`，详细分析请见后面的`第3.1小节`；
4. **给`initializers`属性赋值**，`initializers`属性为`List<ApplicationContextInitializer<?>>`集合，利用SpringBoot的SPI机制从`spring.factories`配置文件中加载，后面在初始化容器的时候会应用这些初始化器来执行一些初始化工作。因为SpringBoot自己实现的SPI机制比较重要，因此独立成一小节来分析，详细分析请见后面的`第4小节`；
5. **给`listeners`属性赋值**，`listeners`属性为`List<ApplicationListener<?>>`集合，同样利用利用SpringBoot的SPI机制从`spring.factories`配置文件中加载。因为SpringBoot启动过程中会在不同的阶段发射一些事件，所以这些加载的监听器们就是来监听SpringBoot启动过程中的一些生命周期事件的；
6. **给`mainApplicationClass`属性赋值**，`mainApplicationClass`属性表示包含`main`函数的类，即这里要推断哪个类调用了`main`函数，然后把这个类的全限定名赋值给`mainApplicationClass`属性，用于后面启动流程中打印一些日志，详细分析见后面的`第3.2小节`。

# 3.1 推断项目应用类型

我们接着分析构造`SpringApplication`对象的第`【3】`步`WebApplicationType.deduceFromClasspath();`这句代码：

```java
// WebApplicationType.java

public enum WebApplicationType {
        // 普通的应用
	NONE,
	// Servlet类型的web应用
	SERVLET,
	// Reactive类型的web应用
	REACTIVE;

	private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };
	private static final String WEBMVC_INDICATOR_CLASS = "org.springframework."
			+ "web.servlet.DispatcherServlet";
	private static final String WEBFLUX_INDICATOR_CLASS = "org."
			+ "springframework.web.reactive.DispatcherHandler";
	private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";
	private static final String SERVLET_APPLICATION_CONTEXT_CLASS = "org.springframework.web.context.WebApplicationContext";
	private static final String REACTIVE_APPLICATION_CONTEXT_CLASS = "org.springframework.boot.web.reactive.context.ReactiveWebApplicationContext";

	static WebApplicationType deduceFromClasspath() {
		// 若classpath中不存在"org.springframework." + "web.servlet.DispatcherServlet"和"org.glassfish.jersey.servlet.ServletContainer"
		// 则返回WebApplicationType.REACTIVE，表明是reactive应用
		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		// 若{ "javax.servlet.Servlet",
		//       "org.springframework.web.context.ConfigurableWebApplicationContext" }
		// 都不存在在classpath，则说明是不是web应用
		for (String className : SERVLET_INDICATOR_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		// 最终返回普通的web应用
		return WebApplicationType.SERVLET;
	}
}
```

如上代码，根据`classpath`判断应用类型，即通过反射加载`classpath`判断指定的标志类存在与否来分别判断是`Reactive`应用，`Servlet`类型的web应用还是普通的应用。

# 3.2 推断哪个类调用了main函数

我们先跳过构造`SpringApplication`对象的第`【4】`步和第`【5】`步，先来分析构造`SpringApplication`对象的第`【6】`步`this.mainApplicationClass = deduceMainApplicationClass();`这句代码：

```java
// SpringApplication.java

private Class<?> deduceMainApplicationClass() {
	try {
		// 获取StackTraceElement对象数组stackTrace，StackTraceElement对象存储了调用栈相关信息（比如类名，方法名等）
		StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
		// 遍历stackTrace数组
		for (StackTraceElement stackTraceElement : stackTrace) {
			// 若stackTraceElement记录的调用方法名等于main
			if ("main".equals(stackTraceElement.getMethodName())) {
				// 那么就返回stackTraceElement记录的类名即包含main函数的类名
				return Class.forName(stackTraceElement.getClassName());
			}
		}
	}
	catch (ClassNotFoundException ex) {
		// Swallow and continue
	}
	return null;
}
```

可以看到`deduceMainApplicationClass`方法的主要作用就是从`StackTraceElement`调用栈数组中获取哪个类调用了`main`方法，然后再返回赋值给`mainApplicationClass`属性，然后用于后面启动流程中打印一些日志。

# 4 SpringBoot的SPI机制原理解读

由于SpringBoot的SPI机制是一个很重要的知识点，因此这里单独一小节来分析。我们都知道，SpringBoot没有使用Java的SPI机制(Java的SPI机制可以看看笔者的[Java是如何实现自己的SPI机制的？](https://juejin.im/post/5e7c26a76fb9a009a441757c),真的是干货满满)，而是自定义实现了一套自己的SPI机制。SpringBoot利用自定义实现的SPI机制可以加载初始化器实现类，监听器实现类和自动配置类等等。如果我们要添加自动配置类或自定义监听器，那么我们很重要的一步就是在`spring.factories`中进行配置，然后才会被SpringBoot加载。

好了，那么接下来我们就来重点分析下**SpringBoot是如何是实现自己的SPI机制的**。

这里接第3小节的构造`SpringApplication`对象的第`【4】`步和第`【5】`步代码，因为第`【4】`步和第`【5】`步都是利用SpringBoot的SPI机制来加载扩展实现类，因此这里只分析第`【4】`步的`setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));`这句代码，看看`getSpringFactoriesInstances`方法中SpringBoot是如何实现自己的一套SPI来加载`ApplicationContextInitializer`初始化器接口的扩展实现类的？

```java
// SpringApplication.java

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    // 继续调用重载的getSpringFactoriesInstances方法进行加载
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}
```

继续跟进重载的`getSpringFactoriesInstances`方法：

```java
// SpringApplication.java

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
		Class<?>[] parameterTypes, Object... args) {
	// 【1】获得类加载器
	ClassLoader classLoader = getClassLoader();
	// Use names and ensure unique to protect against duplicates
	// 【2】将接口类型和类加载器作为参数传入loadFactoryNames方法，从spring.factories配置文件中进行加载接口实现类
	Set<String> names = new LinkedHashSet<>(
			SpringFactoriesLoader.loadFactoryNames(type, classLoader));
	// 【3】实例化从spring.factories中加载的接口实现类
	List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
			classLoader, args, names);
	// 【4】进行排序
	AnnotationAwareOrderComparator.sort(instances);
	// 【5】返回加载并实例化好的接口实现类
	return instances;
}
```

可以看到，SpringBoot自定义实现的SPI机制代码中最重要的是上面代码的`【1】`,`【2】`,`【3】`步，这3步下面分别进行重点分析。

# 4.1 获得类加载器

还记得[Java是如何实现自己的SPI机制的？](https://juejin.im/post/5e7c26a76fb9a009a441757c)这篇文章中Java的SPI机制默认是利用线程上下文类加载器去加载扩展类的，那么，**SpringBoot自己实现的SPI机制又是利用哪种类加载器去加载`spring.factories`配置文件中的扩展实现类呢？**

我们直接看第`【1】`步的`ClassLoader classLoader = getClassLoader();`这句代码，先睹为快：

```java
// SpringApplication.java

public ClassLoader getClassLoader() {
	// 前面在构造SpringApplicaiton对象时，传入的resourceLoader参数是null，因此不会执行if语句里面的逻辑
	if (this.resourceLoader != null) {
		return this.resourceLoader.getClassLoader();
	}
	// 获取默认的类加载器
	return ClassUtils.getDefaultClassLoader();
}
```

继续跟进`getDefaultClassLoader`方法：

```java
// ClassUtils.java

public static ClassLoader getDefaultClassLoader() {
	ClassLoader cl = null;
	try {
	        // 【重点】获取线程上下文类加载器
		cl = Thread.currentThread().getContextClassLoader();
	}
	catch (Throwable ex) {
		// Cannot access thread context ClassLoader - falling back...
	}
	// 这里的逻辑不会执行
	if (cl == null) {
		// No thread context class loader -> use class loader of this class.
		cl = ClassUtils.class.getClassLoader();
		if (cl == null) {
			// getClassLoader() returning null indicates the bootstrap ClassLoader
			try {
				cl = ClassLoader.getSystemClassLoader();
			}
			catch (Throwable ex) {
				// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
			}
		}
	}
	// 返回刚才获取的线程上下文类加载器
	return cl;
}
```

可以看到，原来SpringBoot的SPI机制中也是用线程上下文类加载器去加载`spring.factories`文件中的扩展实现类的！

# 4.2 加载spring.factories配置文件中的SPI扩展类

我们再来看下第`【2】`步中的`SpringFactoriesLoader.loadFactoryNames(type, classLoader)`这句代码是如何加载`spring.factories`配置文件中的SPI扩展类的？

```java
// SpringFactoriesLoader.java

public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
        // factoryClass即SPI接口，比如ApplicationContextInitializer,EnableAutoConfiguration等接口
	String factoryClassName = factoryClass.getName();
	// 【主线，重点关注】继续调用loadSpringFactories方法加载SPI扩展类
	return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}
```

继续跟进`loadSpringFactories`方法：

```java
// SpringFactoriesLoader.java

/**
 * The location to look for factories.
 * <p>Can be present in multiple JAR files.
 */
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
	// 以classLoader作为键先从缓存中取，若能取到则直接返回
	MultiValueMap<String, String> result = cache.get(classLoader);
	if (result != null) {
		return result;
	}
	// 若缓存中无记录，则去spring.factories配置文件中获取
	try {
		// 这里加载所有jar包中包含"MATF-INF/spring.factories"文件的url路径
		Enumeration<URL> urls = (classLoader != null ?
				classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
				ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
		result = new LinkedMultiValueMap<>();
		// 遍历urls路径，将所有spring.factories文件的键值对（key:SPI接口类名 value:SPI扩展类名）
		// 加载放到 result集合中
		while (urls.hasMoreElements()) {
			// 取出一条url
			URL url = urls.nextElement();
			// 将url封装到UrlResource对象中
			UrlResource resource = new UrlResource(url);
			// 利用PropertiesLoaderUtils的loadProperties方法将spring.factories文件键值对内容加载进Properties对象中
			Properties properties = PropertiesLoaderUtils.loadProperties(resource);
			// 遍历刚加载的键值对properties对象
			for (Map.Entry<?, ?> entry : properties.entrySet()) {
				// 取出SPI接口名
				String factoryClassName = ((String) entry.getKey()).trim();
				// 遍历SPI接口名对应的实现类即SPI扩展类
				for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
					// SPI接口名作为key，SPI扩展类作为value放入result中
					result.add(factoryClassName, factoryName.trim());
				}
			}
		}
		// 以classLoader作为key，result作为value放入cache缓存
		cache.put(classLoader, result);
		// 最终返回result对象
		return result;
	}
	catch (IOException ex) {
		throw new IllegalArgumentException("Unable to load factories from location [" +
				FACTORIES_RESOURCE_LOCATION + "]", ex);
	}
}
```

如上代码，`loadSpringFactories`方法主要做的事情就是利用之前获取的线程上下文类加载器将`classpath`中的所有`spring.factories`配置文件中所有SPI接口的所有扩展实现类给加载出来，然后放入缓存中。**注意**，这里是一次性加载所有的SPI扩展实现类哈，所以之后根据SPI接口就直接从缓存中获取SPI扩展类了，就不用再次去`spring.factories`配置文件中获取SPI接口对应的扩展实现类了。比如之后的获取`ApplicationListener`,`FailureAnalyzer`和`EnableAutoConfiguration`接口的扩展实现类都直接从缓存中获取即可。

> **思考1：** 这里为啥要一次性从`spring.factories`配置文件中获取所有的扩展类放入缓存中呢？而不是每次都是根据SPI接口去`spring.factories`配置文件中获取呢？

> **思考2：** 还记得之前讲的SpringBoot的自动配置源码时提到的`AutoConfigurationImportFilter`这个接口的作用吗？现在我们应该能更清楚的理解这个接口的作用了吧。


将所有的SPI扩展实现类加载出来后，此时再调用`getOrDefault(factoryClassName, Collections.emptyList())`方法根据SPI接口名去筛选当前对应的扩展实现类，比如这里传入的`factoryClassName`参数名为`ApplicationContextInitializer`接口，那么这个接口将会作为`key`从刚才缓存数据中取出`ApplicationContextInitializer`接口对应的SPI扩展实现类。其中从`spring.factories`中获取的`ApplicationContextInitializer`接口对应的所有SPI扩展实现类如下图所示：

![](https://user-gold-cdn.xitu.io/2020/3/31/1712c91d81a89901?w=713&h=199&f=png&s=21757)

# 4.3 实例化从spring.factories中加载的SPI扩展类

前面从`spring.factories`中获取到`ApplicationContextInitializer`接口对应的所有SPI扩展实现类后，此时会将这些SPI扩展类进行实例化。

此时我们再来看下前面的第`【3】`步的实例化代码：
`List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);`。

```java
// SpringApplication.java

private <T> List<T> createSpringFactoriesInstances(Class<T> type,
		Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
		Set<String> names) {
	// 新建instances集合，用于存储稍后实例化后的SPI扩展类对象
	List<T> instances = new ArrayList<>(names.size());
	// 遍历name集合，names集合存储了所有SPI扩展类的全限定名
	for (String name : names) {
		try {
			// 根据全限定名利用反射加载类
			Class<?> instanceClass = ClassUtils.forName(name, classLoader);
			// 断言刚才加载的SPI扩展类是否属于SPI接口类型
			Assert.isAssignable(type, instanceClass);
			// 获得SPI扩展类的构造器
			Constructor<?> constructor = instanceClass
					.getDeclaredConstructor(parameterTypes);
			// 实例化SPI扩展类
			T instance = (T) BeanUtils.instantiateClass(constructor, args);
			// 添加进instances集合
			instances.add(instance);
		}
		catch (Throwable ex) {
			throw new IllegalArgumentException(
					"Cannot instantiate " + type + " : " + name, ex);
		}
	}
	// 返回
	return instances;
}
```

上面代码很简单，主要做的事情就是实例化SPI扩展类。
好了，SpringBoot自定义的SPI机制就已经分析完了。

> **思考3：** SpringBoot为何弃用Java的SPI而自定义了一套SPI？

# 5 小结

好了，本片就到此结束了，先将前面的知识点再总结下：

1. 分析了`SpringApplication`对象的构造过程；
2. 分析了SpringBoot自己实现的一套SPI机制。

# 6 有感而发

从自己2月开始写源码分析文章以来，也认识了一些技术大牛，从他们身上看到，越厉害的人越努力。回想一下，自己现在知识面也很窄，更重要的是对自己所涉及的技术没有深度，一句话概括，还很菜，而看到比自己厉害的大牛们都还那么拼，自己有啥理由不努力呢？很喜欢丁威老师的一句话："**唯有坚持不懈**"。然后自己一步一个脚印，相信自己能取得更大的进步，继续加油。



**点赞和转发是对笔者最大的激励哦！**

由于笔者水平有限，若文中有错误还请指出，谢谢。

---------------------------------------------------

公众号【源码笔记】，专注于Java后端系列框架的源码分析。
![](https://user-gold-cdn.xitu.io/2020/3/15/170dd9bb2b5b59de?w=142&h=135&f=png&s=39743)