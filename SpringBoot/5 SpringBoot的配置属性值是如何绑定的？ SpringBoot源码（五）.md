**注：该源码分析对应SpringBoot版本为2.1.0.RELEASE**

# 1 前言

本篇接 [SpringBoot是如何实现自动配置的？--SpringBoot源码（四）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/4%20SpringBoot%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE%E7%9A%84%EF%BC%9F%20%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E5%9B%9B%EF%BC%89.md)

温故而知新，我们来简单回顾一下上篇的内容，上一篇我们分析了SpringBoot的自动配置的相关源码，自动配置相关源码主要有以下几个重要的步骤：

1. 从spring.factories配置文件中加载自动配置类；

2. 加载的自动配置类中排除掉`@EnableAutoConfiguration`注解的`exclude`属性指定的自动配置类；

3. 然后再用`AutoConfigurationImportFilter`接口去过滤自动配置类是否符合其标注注解（若有标注的话）`@ConditionalOnClass`,`@ConditionalOnBean`和`@ConditionalOnWebApplication`的条件，若都符合的话则返回匹配结果；

4. 然后触发`AutoConfigurationImportEvent`事件，告诉`ConditionEvaluationReport`条件评估报告器对象来分别记录符合条件和`exclude`的自动配置类。

5. 最后spring再将最后筛选后的自动配置类导入IOC容器中

   

本篇继续来分析SpringBoot的自动配置的相关源码，我们来分析下`@EnableConfigurationProperties`和`@EnableConfigurationProperties`这两个注解，来探究下**外部配置属性值是如何被绑定到@ConfigurationProperties注解的类属性中的？**

> 举个栗子：以配置web项目的服务器端口为例，若我们要将服务器端口配置为`8081`，那么我们会在`application.properties`配置文件中配置`server.port=8081`，此时该配置值`8081`就将会绑定到被`@ConfigurationProperties`注解的类`ServerProperties`的属性`port`上，从而使得配置生效。





# 2 @EnableConfigurationProperties



我们接着前面的设置服务器端口的栗子来分析，我们先直接来看看`ServerProperties`的源码，应该能找到源码的入口：

```java
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties {
	/**
	 * Server HTTP port.
	 */
	private Integer port;
	// ...省略非关键代码
}
```

可以看到，`ServerProperties`类上标注了`@ConfigurationProperties`这个注解，服务器属性配置前缀为`server`，是否忽略未知的配置值（`ignoreUnknownFields`）设置为`true`。

那么我们再来看下`@ConfigurationProperties`这个注解的源码：

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ConfigurationProperties {
    
	// 前缀别名
	@AliasFor("prefix")
	String value() default "";
    
	// 前缀
	@AliasFor("value")
	String prefix() default "";
    
	// 忽略无效的配置属性
	boolean ignoreInvalidFields() default false;
    
	// 忽略未知的配置属性
	boolean ignoreUnknownFields() default true;
}
```

`@ConfigurationProperties`这个注解的作用就是将外部配置的配置值绑定到其注解的类的属性上，可以作用于配置类或配置类的方法上。可以看到`@ConfigurationProperties`注解除了有设置前缀，是否忽略一些不存在或无效的配置等属性等外，这个注解没有其他任何的处理逻辑，可以看到`@ConfigurationProperties`是一个标志性的注解，**源码入口不在这里**。

这里讲的是服务器的自动配置，自然而然的，我们来看下自动配置类`ServletWebServerFactoryAutoConfiguration`的源码：

```java
@Configuration
@EnableConfigurationProperties(ServerProperties.class)
// ...省略非关键注解
public class ServletWebServerFactoryAutoConfiguration {
	// ...省略非关键代码
}
```

为了突出重点，我已经把`ServletWebServerFactoryAutoConfiguration`的非关键代码和非关键注解省略掉了。可以看到，`ServletWebServerFactoryAutoConfiguration`自动配置类中有一个`@EnableConfigurationProperties`注解，且注解值是前面讲的`ServerProperties.class`，因此`@EnableConfigurationProperties`注解肯定就是我们关注的重点了。

同样，再来看下`@EnableConfigurationProperties`注解的源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EnableConfigurationPropertiesImportSelector.class)
public @interface EnableConfigurationProperties {

	// 这个值指定的类就是@ConfigurationProperties注解标注的类，其将会被注册到spring容器中
	Class<?>[] value() default {};

}
```

`@EnableConfigurationProperties`注解的主要作用就是为`@ConfigurationProperties`注解标注的类提供支持，即对将外部配置属性值（比如application.properties配置值）绑定到`@ConfigurationProperties`标注的类的属性中。
> **注意**：SpringBoot源码中还存在了`ConfigurationPropertiesAutoConfiguration`这个自动配置类，同时`spring.factories`配置文件中的`EnableAutoConfiguration`接口也配置了`ConfigurationPropertiesAutoConfiguration`，这个自动配置类上也有`@EnableConfigurationProperties`这个注解，堆属性绑定进行了默认开启。

**那么，`@EnableConfigurationProperties`这个注解对属性绑定提供怎样的支持呢？**

可以看到`@EnableConfigurationProperties`这个注解上还标注了`@Import(EnableConfigurationPropertiesImportSelector.class)`，其导入了`EnableConfigurationPropertiesImportSelector`，因此可以肯定的是`@EnableConfigurationProperties`这个注解对属性绑定提供的支持必定跟`EnableConfigurationPropertiesImportSelector`有关。

到了这里，`EnableConfigurationPropertiesImportSelector`这个哥们是我们接下来要分析的对象，那么我们下面继续来分析`EnableConfigurationPropertiesImportSelector`是如何承担将外部配置属性值绑定到`@ConfigurationProperties`标注的类的属性中的。



# 3 EnableConfigurationPropertiesImportSelector
`EnableConfigurationPropertiesImportSelector`类的作用主要用来处理外部属性绑定的相关逻辑，其实现了`ImportSelector`接口，我们都知道，实现`ImportSelector`接口的`selectImports`方法可以向容器中注册bean。

那么，我们来看下`EnableConfigurationPropertiesImportSelector`覆写的`selectImports`方法：
```java
// EnableConfigurationPropertiesImportSelector.java

class EnableConfigurationPropertiesImportSelector implements ImportSelector {
        // IMPORTS数组即是要向spring容器中注册的bean
	private static final String[] IMPORTS = {
			ConfigurationPropertiesBeanRegistrar.class.getName(),
			ConfigurationPropertiesBindingPostProcessorRegistrar.class.getName() };

	@Override
	public String[] selectImports(AnnotationMetadata metadata) {
		// 返回ConfigurationPropertiesBeanRegistrar和ConfigurationPropertiesBindingPostProcessorRegistrar的全限定名
		// 即上面两个类将会被注册到Spring容器中
		return IMPORTS;
	}
	
}
```
可以看到`EnableConfigurationPropertiesImportSelector`类中的`selectImports`方法中返回的是`IMPORTS`数组，而这个`IMPORTS`是一个常量数组，值是`ConfigurationPropertiesBeanRegistrar`和`ConfigurationPropertiesBindingPostProcessorRegistrar`。即`EnableConfigurationPropertiesImportSelector`的作用是向Spring容器中注册了`ConfigurationPropertiesBeanRegistrar`和`ConfigurationPropertiesBindingPostProcessorRegistrar`这两个`bean`。

我们在`EnableConfigurationPropertiesImportSelector`类中没看到处理外部属性绑定的相关逻辑，其只是注册了`ConfigurationPropertiesBeanRegistrar`和`ConfigurationPropertiesBindingPostProcessorRegistrar`这两个`bean`,接下来我们再看下注册的这两个`bean`类。
# 4 ConfigurationPropertiesBeanRegistrar
我们先来看下`ConfigurationPropertiesBeanRegistrar`这个类。

`ConfigurationPropertiesBeanRegistrar`是`EnableConfigurationPropertiesImportSelector`的内部类，其实现了`ImportBeanDefinitionRegistrar`接口，覆写了`registerBeanDefinitions`方法。可见，`ConfigurationPropertiesBeanRegistrar`又是用来注册一些`bean` `definition`的，即也是向`Spring`容器中注册一些bean。

先看下`ConfigurationPropertiesBeanRegistrar`的源码：
```java
// ConfigurationPropertiesBeanRegistrar$ConfigurationPropertiesBeanRegistrar.java

public static class ConfigurationPropertiesBeanRegistrar
			implements ImportBeanDefinitionRegistrar {
	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,  // metadata是AnnotationMetadataReadingVisitor对象，存储了某个配置类的元数据
			BeanDefinitionRegistry registry) {
		// （1）得到@EnableConfigurationProperties注解的所有属性值,
		// 比如@EnableConfigurationProperties(ServerProperties.class),那么得到的值是ServerProperties.class
		// （2）然后再将得到的@EnableConfigurationProperties注解的所有属性值注册到容器中
		getTypes(metadata).forEach((type) -> register(registry,
				(ConfigurableListableBeanFactory) registry, type));
	}
}
```
在`ConfigurationPropertiesBeanRegistrar`实现的`registerBeanDefinitions`中，可以看到主要做了两件事：
1. 调用`getTypes`方法获取`@EnableConfigurationProperties`注解的属性值`XxxProperties`；
2. 调用`register`方法将获取的属性值`XxxProperties`注册到`Spring`容器中，用于以后和外部属性绑定时使用。

我们来看下`getTypes`方法的源码：
```java
// ConfigurationPropertiesBeanRegistrar$ConfigurationPropertiesBeanRegistrar.java

private List<Class<?>> getTypes(AnnotationMetadata metadata) {
	// 得到@EnableConfigurationProperties注解的所有属性值,
	// 比如@EnableConfigurationProperties(ServerProperties.class),那么得到的值是ServerProperties.class
	MultiValueMap<String, Object> attributes = metadata
			.getAllAnnotationAttributes(
					EnableConfigurationProperties.class.getName(), false);
	// 将属性值取出装进List集合并返回
	return collectClasses((attributes != null) ? attributes.get("value")
			: Collections.emptyList());
}
```
`getTypes`方法里面的逻辑很简单即将`@EnableConfigurationProperties`注解里面的属性值`XxxProperties`（比如`ServerProperties.class`）取出并装进`List`集合并返回。

由`getTypes`方法拿到`@EnableConfigurationProperties`注解里面的属性值`XxxProperties`（比如`ServerProperties.class`）后，此时再遍历将`XxxProperties`逐个注册进`Spring`容器中，我们来看下`register`方法：
```java
// ConfigurationPropertiesBeanRegistrar$ConfigurationPropertiesBeanRegistrar.java

private void register(BeanDefinitionRegistry registry,
		ConfigurableListableBeanFactory beanFactory, Class<?> type) {
	// 得到type的名字，一般用类的全限定名作为bean name
	String name = getName(type);
	// 根据bean name判断beanFactory容器中是否包含该bean
	if (!containsBeanDefinition(beanFactory, name)) {
		// 若不包含，那么注册bean definition
		registerBeanDefinition(registry, name, type);
	}
}
```

我们再来看下由`EnableConfigurationPropertiesImportSelector`导入的另一个类`ConfigurationPropertiesBindingPostProcessorRegistrar`又是干嘛的呢？
# 5 ConfigurationPropertiesBindingPostProcessorRegistrar
可以看到`ConfigurationPropertiesBindingPostProcessorRegistrar`类名字又是以`Registrar`单词为结尾，说明其肯定又是导入一些`bean` `definition`的。直接看源码：
```java
// ConfigurationPropertiesBindingPostProcessorRegistrar.java

public class ConfigurationPropertiesBindingPostProcessorRegistrar
		implements ImportBeanDefinitionRegistrar {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
			BeanDefinitionRegistry registry) {
		// 若容器中没有注册ConfigurationPropertiesBindingPostProcessor这个处理属性绑定的后置处理器，
		// 那么将注册ConfigurationPropertiesBindingPostProcessor和ConfigurationBeanFactoryMetadata这两个bean
		// 注意onApplicationEnvironmentPreparedEvent事件加载配置属性在先，然后再注册一些后置处理器用来处理这些配置属性
		if (!registry.containsBeanDefinition(
				ConfigurationPropertiesBindingPostProcessor.BEAN_NAME)) {
			// (1)注册ConfigurationPropertiesBindingPostProcessor后置处理器，用来对配置属性进行后置处理
			registerConfigurationPropertiesBindingPostProcessor(registry);
			// (2)注册一个ConfigurationBeanFactoryMetadata类型的bean，
			// 注意ConfigurationBeanFactoryMetadata实现了BeanFactoryPostProcessor，然后其会在postProcessBeanFactory中注册一些元数据
			registerConfigurationBeanFactoryMetadata(registry);
		}
	}
	// 注册ConfigurationPropertiesBindingPostProcessor后置处理器
	private void registerConfigurationPropertiesBindingPostProcessor(
			BeanDefinitionRegistry registry) {
		GenericBeanDefinition definition = new GenericBeanDefinition();
		definition.setBeanClass(ConfigurationPropertiesBindingPostProcessor.class);
		definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(
				ConfigurationPropertiesBindingPostProcessor.BEAN_NAME, definition);

	}
	// 注册ConfigurationBeanFactoryMetadata后置处理器
	private void registerConfigurationBeanFactoryMetadata(
			BeanDefinitionRegistry registry) {
		GenericBeanDefinition definition = new GenericBeanDefinition();
		definition.setBeanClass(ConfigurationBeanFactoryMetadata.class);
		definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(ConfigurationBeanFactoryMetadata.BEAN_NAME,
				definition);
	}

}
```


`ConfigurationPropertiesBindingPostProcessorRegistrar`类的逻辑非常简单，主要用来注册外部配置属性绑定相关的后置处理器即`ConfigurationBeanFactoryMetadata`和`ConfigurationPropertiesBindingPostProcessor`。

那么接下来我们再来探究下注册的这两个后置处理器又是执行怎样的后置处理逻辑呢？
# 6 ConfigurationBeanFactoryMetadata
先来看`ConfigurationBeanFactoryMetadata`这个后置处理器，其实现了`BeanFactoryPostProcessor`接口的`postProcessBeanFactory`方法，在初始化`bean` `factory`时将`@Bean`注解的元数据存储起来，以便在后续的外部配置属性绑定的相关逻辑中使用。

先来看下`ConfigurationBeanFactoryMetadata`类实现`BeanFactoryPostProcessor`接口的`postProcessBeanFactory`方法源码：
```java
// ConfigurationBeanFactoryMetadata

public class ConfigurationBeanFactoryMetadata implements BeanFactoryPostProcessor {

	/**
	 * The bean name that this class is registered with.
	 */
	public static final String BEAN_NAME = ConfigurationBeanFactoryMetadata.class
			.getName();

	private ConfigurableListableBeanFactory beanFactory;
	/**
	 * beansFactoryMetadata集合存储beansFactory的元数据
	 * key:某个bean的名字  value：FactoryMetadata对象（封装了工厂bean名和工厂方法名）
	 * 比如下面这个配置类：
	 *
	 * @Configuration
	 * public class ConfigA {
	 *      @Bean
	 *      public BeanXXX methodB（configA, ） {
	 *          return new BeanXXX();
	 *      }
	 * }
	 *
	 * 那么：key值为"methodB"，value为FactoryMetadata（configA, methodB）对象，其bean属性值为"configA",method属性值为"methodB"
	 */
	private final Map<String, FactoryMetadata> beansFactoryMetadata = new HashMap<>();

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
			throws BeansException {
		this.beanFactory = beanFactory;
		// 遍历beanFactory的beanDefinitionName，即每个bean的名字（比如工厂方法对应的bean名字）
		for (String name : beanFactory.getBeanDefinitionNames()) {
			// 根据name得到beanDefinition
			BeanDefinition definition = beanFactory.getBeanDefinition(name);
			// 工厂方法名：一般是注解@Bean的方法名
			String method = definition.getFactoryMethodName();
			// 工厂bean名：一般是注解@Configuration的类名
			String bean = definition.getFactoryBeanName();
			if (method != null && bean != null) {
				// 将beanDefinitionName作为Key，封装了工厂bean名和工厂方法名的FactoryMetadata对象作为value装入beansFactoryMetadata中
				this.beansFactoryMetadata.put(name, new FactoryMetadata(bean, method));
			}
		}
	}
}
```
从上面代码可以看到`ConfigurationBeanFactoryMetadata`类覆写的`postProcessBeanFactory`方法做的事情就是将工厂`Bean`（可以理解为`@Configuration`注解的类）及其`@Bean`注解的工厂方法的一些元数据缓存到`beansFactoryMetadata`集合中，以便后续使用，这个后面会详述。

由上代码中我们看到了`ConfigurationBeanFactoryMetadata`类的`beansFactoryMetadata`集合类型是`Map<String, FactoryMetadata>`，那么我们再来看下封装相关工厂元数据的`FactoryMetadata`类：
```java
// ConfigurationBeanFactoryMetadata$FactoryMetadata.java

private static class FactoryMetadata {
	// @Configuration注解的配置类的类名
	private final String bean;
	// @Bean注解的方法名
	private final String method;

	FactoryMetadata(String bean, String method) {
		this.bean = bean;
		this.method = method;
	}

	public String getBean() {
		return this.bean;
	}

	public String getMethod() {
		return this.method;
	}

}
```
`FactoryMetadata`仅有两个属性`bean`和`method`,分别表示`@Configuration`注解的工厂`bean`和`@Bean`注解的工厂方法。

上面说了那么多，直接举个栗子会更直观：
```java
/**
 * beansFactoryMetadata集合存储beansFactory的元数据
 * key:某个bean的名字  value：FactoryMetadata对象（封装了工厂bean名和工厂方法名）
 * 比如下面这个配置类：
 *
 * @Configuration
 * public class ConfigA {
 *      @Bean
 *      public BeanXXX methodB（configA, ） {
 *          return new BeanXXX();
 *      }
 * }
 *
 * 那么：key值为"methodB"，value为FactoryMetadata（configA, methodB）对象，其bean属性值为"configA",method属性值为"methodB"
 */
 private final Map<String, FactoryMetadata> beansFactoryMetadata = new HashMap<>();
```
为了更好理解上面`beansFactoryMetadata`集合存储的数据是啥，建议最好自己动手调试看看其里面装的是什么哦。总之这里记住一点就好了：`ConfigurationBeanFactoryMetadata`类的`beansFactoryMetadata`集合存储的是工厂`bean`的相关元数据，以便在`ConfigurationPropertiesBindingPostProcessor`后置处理器中使用。




# 7 ConfigurationPropertiesBindingPostProcessor
我们再来看下`ConfigurationPropertiesBindingPostProcessorRegistrar`类注册的另外一个后置处理器`ConfigurationPropertiesBindingPostProcessor`，这个后置处理器就**尤其重要**了，主要承担了**将外部配置属性绑定到`@ConfigurationProperties`注解标注的XxxProperties类的属性中**（比如`application.properties`配置文件中设置了`server.port=8081`,那么`8081`将会绑定到`ServerProperties`类的`port`属性中）的实现逻辑。

同样，先来看下`ConfigurationPropertiesBindingPostProcessor`的源码：
```java
// ConfigurationPropertiesBindingPostProcessor.java

public class ConfigurationPropertiesBindingPostProcessor implements BeanPostProcessor,
	PriorityOrdered, ApplicationContextAware, InitializingBean {
	@Override
	public void afterPropertiesSet() throws Exception {
	    // ...这里省略实现代码先
	}
	
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
	    // ...这里省略实现代码先
	}
	
	// ...省略非关键代码
}
```

可以看到`ConfigurationPropertiesBindingPostProcessor`后置处理器实现了两个重要的接口`InitializingBean`和`BeanPostProcessor`。
> 我们都知道：
1. `InitializingBean`接口的`afterPropertiesSet`方法会在`bean`属性赋值后调用，用来执行一些自定义的初始化逻辑比如检查某些强制的属性是否有被赋值，校验某些配置或给一些未被赋值的属性赋值。
2. `BeanPostProcessor`接口是`bean`的后置处理器，其有`postProcessBeforeInitialization`和`postProcessAfterInitialization`两个勾子方法，分别会在`bean`初始化前后被调用来执行一些后置处理逻辑，比如检查标记接口或是否用代理包装了`bean`。

同时由上代码可以看到`ConfigurationPropertiesBindingPostProcessor`后置处理器覆写了`InitializingBean`的`afterPropertiesSet`方法和`BeanPostProcessor`的`postProcessBeforeInitialization`方法。

接下来我们再来探究`ConfigurationPropertiesBindingPostProcessor`后置处理器覆写的两个方法的源码。

# 7.1 在执行外部属性绑定逻辑前先准备好相关元数据和配置属性绑定器
我们先来分析下`ConfigurationPropertiesBindingPostProcessor`覆写`InitializingBean`接口的`afterPropertiesSet`方法：
```java
// ConfigurationPropertiesBindingPostProcessor.java

        /**
	 * 配置属性校验器名字
	 */
	public static final String VALIDATOR_BEAN_NAME = "configurationPropertiesValidator";
	/**
	 * 工厂bean相关元数据
	 */
	private ConfigurationBeanFactoryMetadata beanFactoryMetadata;
	/**
	 * 上下文
	 */
	private ApplicationContext applicationContext;
	/**
	 * 配置属性绑定器
	 */
	private ConfigurationPropertiesBinder configurationPropertiesBinder;
	
	
    // 这里主要是给beanFactoryMetadata和configurationPropertiesBinder的属性赋值，用于后面的后置处理器方法处理属性绑定的时候用
	@Override
	public void afterPropertiesSet() throws Exception {
		// We can't use constructor injection of the application context because
		// it causes eager factory bean initialization
		// 【1】利用afterPropertiesSet这个勾子方法从容器中获取之前注册的ConfigurationBeanFactoryMetadata对象赋给beanFactoryMetadata属性
		// （问1）beanFactoryMetadata这个bean是什么时候注册到容器中的？
		// （答1）在ConfigurationPropertiesBindingPostProcessorRegistrar类的registerBeanDefinitions方法中将beanFactoryMetadata这个bean注册到容器中
		// （问2）从容器中获取beanFactoryMetadata对象后，什么时候会被用到？
		// （答2）beanFactoryMetadata对象的beansFactoryMetadata集合保存的工厂bean相关的元数据，在ConfigurationPropertiesBindingPostProcessor类
		//        要判断某个bean是否有FactoryAnnotation或FactoryMethod时会根据这个beanFactoryMetadata对象的beansFactoryMetadata集合的元数据来查找
		this.beanFactoryMetadata = this.applicationContext.getBean(
				ConfigurationBeanFactoryMetadata.BEAN_NAME,
				ConfigurationBeanFactoryMetadata.class);
		// 【2】new一个ConfigurationPropertiesBinder，用于后面的外部属性绑定时使用
		this.configurationPropertiesBinder = new ConfigurationPropertiesBinder(
				this.applicationContext, VALIDATOR_BEAN_NAME); // VALIDATOR_BEAN_NAME="configurationPropertiesValidator"
	}

```

可以看到以上代码主要逻辑就是**在执行外部属性绑定逻辑前先准备好相关元数据和配置属性绑定器**，即从`Spring`容器中获取到之前注册的`ConfigurationBeanFactoryMetadata`对象赋给`ConfigurationPropertiesBindingPostProcessor`后置处理器的`beanFactoryMetadata`属性,还有就是新建一个`ConfigurationPropertiesBinder`配置属性绑定器对象并赋值给`configurationPropertiesBinder`属性。

我们再来看下`ConfigurationPropertiesBinder`这个配置属性绑定器对象是如何构造的。

```java
// ConfigurationPropertiesBinder.java

ConfigurationPropertiesBinder(ApplicationContext applicationContext,
		String validatorBeanName) {
	this.applicationContext = applicationContext;
	// 将applicationContext封装到PropertySourcesDeducer对象中并返回
	this.propertySources = new PropertySourcesDeducer(applicationContext)
			.getPropertySources(); // 获取属性源，主要用于在ConfigurableListableBeanFactory的后置处理方法postProcessBeanFactory中处理
	// 如果没有配置validator的话，这里一般返回的是null
	this.configurationPropertiesValidator = getConfigurationPropertiesValidator(
			applicationContext, validatorBeanName);
	// 检查实现JSR-303规范的bean校验器相关类在classpath中是否存在
	this.jsr303Present = ConfigurationPropertiesJsr303Validator
			.isJsr303Present(applicationContext);
}
```
可以看到在构造`ConfigurationPropertiesBinder`对象时主要给其相关属性赋值（一般构造器逻辑都是这样）：
1. 给`applicationContext`属性赋值注入上下文对象；
2. 给`propertySources`属性赋值，属性源即外部配置值比如`application.properties`配置的属性值，注意这里的属性源是由`ConfigFileApplicationListener`这个监听器负责读取的，`ConfigFileApplicationListener`将会在后面源码分析章节中详述。
3. 给`configurationPropertiesValidator`属性赋值，值来自`Spring`容器中名为`configurationPropertiesValidator`的`bean`。
4. 给`jsr303Present`属性赋值，当`javax.validation.Validator`,`javax.validation.ValidatorFactory`和`javax.validation.bootstrap.GenericBootstrap"`这三个类同时存在于`classpath`中`jsr303Present`属性值才为`true`。

> **关于JSR303**：`JSR-303`是JAVA EE 6中的一项子规范，叫做`Bean Validation`，`Hibernate Validator`是`Bean Validation`的参考实现 。`Hibernate Validator`提供了`JSR 303`规范中所有内置`constraint` 的实现，除此之外还有一些附加的`constraint`。

# 7.2 执行真正的外部属性绑定逻辑【主线】
前面分析了那么多，发现都还没到外部属性绑定的真正处理逻辑，前面步骤都是在做一些准备性工作，为外部属性绑定做铺垫。

在执行外部属性绑定逻辑前，准备好了相关元数据和配置属性绑定器后，此时我们再来看看`ConfigurationPropertiesBindingPostProcessor`实现`BeanPostProcessor`接口的`postProcessBeforeInitialization`后置处理方法了，**外部属性绑定逻辑**都是在这个后置处理方法里实现，是我们关注的**重中之重**。

直接看代码：
```java
// ConfigurationPropertiesBindingPostProcessor.java

// 因为是外部配置属性后置处理器，因此这里对@ConfigurationProperties注解标注的XxxProperties类进行后置处理完成属性绑定
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName)
		throws BeansException {
	// 注意，BeanPostProcessor后置处理器默认会对所有的bean进行处理，因此需要根据bean的一些条件进行过滤得到最终要处理的目的bean，
	// 这里的过滤条件就是判断某个bean是否有@ConfigurationProperties注解
	// 【1】从bean上获取@ConfigurationProperties注解,若bean有标注，那么返回该注解；若没有，则返回Null。比如ServerProperty上标注了@ConfigurationProperties注解
	ConfigurationProperties annotation = getAnnotation(bean, beanName,
			ConfigurationProperties.class);
	// 【2】若标注有@ConfigurationProperties注解的bean，那么则进行进一步处理：将配置文件的配置注入到bean的属性值中
	if (annotation != null) {
		/********主线，重点关注】********/
		bind(bean, beanName, annotation); 
	}
	// 【3】返回外部配置属性值绑定后的bean（一般是XxxProperties对象）
	return bean;
}
```
`ConfigurationPropertiesBindingPostProcessor`类覆写的`postProcessBeforeInitialization`方法的做的事情就是将外部属性配置绑定到`@ConfigurationProperties`注解标注的`XxxProperties`类上，现关键步骤总结如下：

1. 从`bean`上获取`@ConfigurationProperties`注解；
2. 若标注有`@ConfigurationProperties`注解的`bean`，那么则进行进一步的处理：将外部配置属性值绑定到bean的属性值中后再返回`bean`；若没有标注有`@ConfigurationProperties`注解的`bean`，那么将直接原样返回`bean`。

>**注意**：后置处理器默认会对每个容器中的`bean`进行后置处理，因为这里只针对标注有`@ConfigurationProperties`注解的`bean`进行外部属性绑定，因此没有标注`@ConfigurationProperties`注解的`bean`将不会被处理。

接下来我们紧跟主线，再来看下**外部配置属性是如何绑定到`@ConfigurationProperties`注解的`XxxProperties`类属性上的呢？**

直接看代码：
```java
// ConfigurationPropertiesBindingPostProcessor.java

private void bind(Object bean, String beanName, ConfigurationProperties annotation) {
	// 【1】得到bean的类型，比如ServerPropertie这个bean得到的类型是：org.springframework.boot.autoconfigure.web.ServerProperties
	ResolvableType type = getBeanType(bean, beanName);
	// 【2】获取bean上标注的@Validated注解
	Validated validated = getAnnotation(bean, beanName, Validated.class);
	// 若标注有@Validated注解的话则跟@ConfigurationProperties注解一起组成一个Annotation数组
	Annotation[] annotations = (validated != null)
			? new Annotation[] { annotation, validated }
			: new Annotation[] { annotation };
	// 【3】返回一个绑定了XxxProperties类的Bindable对象target，这个target对象即被外部属性值注入的目标对象
	// （比如封装了标注有@ConfigurationProperties注解的ServerProperties对象的Bindable对象）
	Bindable<?> target = Bindable.of(type).withExistingValue(bean)
			.withAnnotations(annotations); // 设置annotations属性数组
	try {
		// 【4】执行外部配置属性绑定逻辑
		/********【主线，重点关注】********/
		this.configurationPropertiesBinder.bind(target);
	}
	catch (Exception ex) {
		throw new ConfigurationPropertiesBindException(beanName, bean, annotation,
				ex);
	}
}
```
关键步骤上面代码已经标注`【x】`，这里在继续讲解外部配置属性绑定的主线逻辑(在<font color=blue>8 ConfigurationPropertiesBinder</font>这一小节分析 )前先穿插一个知识点，还记得`ConfigurationBeanFactoryMetadata`覆写的`postProcessBeanFactory`方法里已经将相关工厂`bean`的元数据封装到`ConfigurationBeanFactoryMetadata`类的`beansFactoryMetadata`集合这一回事吗？

我们再来看下上面代码中的`【1】getBeanType`和`【2】getAnnotation`方法源码：
```java
// ConfigurationPropertiesBindingPostProcessor.java

private ResolvableType getBeanType(Object bean, String beanName) {
	// 首先获取有没有工厂方法
	Method factoryMethod = this.beanFactoryMetadata.findFactoryMethod(beanName);
	// 若有工厂方法
	if (factoryMethod != null) { 
		return ResolvableType.forMethodReturnType(factoryMethod);
	} 
	// 没有工厂方法，则说明是普通的配置类
	return ResolvableType.forClass(bean.getClass());
}

private <A extends Annotation> A getAnnotation(Object bean, String beanName,
		Class<A> type) {
	A annotation = this.beanFactoryMetadata.findFactoryAnnotation(beanName, type);
	if (annotation == null) {
		annotation = AnnotationUtils.findAnnotation(bean.getClass(), type);
	}
	return annotation;
}
```
注意到上面代码中的`beanFactoryMetadata`对象没，`ConfigurationPropertiesBindingPostProcessor`后置处理器的`getBeanType`和`getAnnotation`方法分别会调用`ConfigurationBeanFactoryMetadata`的`findFactoryMethod`和`findFactoryAnnotation`方法，而`ConfigurationBeanFactoryMetadata`的`findFactoryMethod`和`findFactoryAnnotation`方法又会依赖存储工厂`bean`元数据的`beansFactoryMetadata`集合来寻找是否有`FactoryMethod`和`FactoryAnnotation`。因此，到这里我们就知道之`ConfigurationBeanFactoryMetadata`的`beansFactoryMetadata`集合存储工厂`bean`元数据的作用了。

# 8 ConfigurationPropertiesBinder
我们再继续紧跟外部配置属性绑定的主线，继续前面看<font color=blue>7.2 执行真正的外部属性绑定逻辑</font>中的`this.configurationPropertiesBinder.bind(target);`这句代码：
```java
// ConfigurationPropertiesBinder.java

public void bind(Bindable<?> target) {
	//【1】得到@ConfigurationProperties注解
	ConfigurationProperties annotation = target
			.getAnnotation(ConfigurationProperties.class);
	Assert.state(annotation != null,
			() -> "Missing @ConfigurationProperties on " + target);
	// 【2】得到Validator对象集合，用于属性校验
	List<Validator> validators = getValidators(target);
	// 【3】得到BindHandler对象（默认是IgnoreTopLevelConverterNotFoundBindHandler对象），
	// 用于对ConfigurationProperties注解的ignoreUnknownFields等属性的处理
	BindHandler bindHandler = getBindHandler(annotation, validators);
	// 【4】得到一个Binder对象，并利用其bind方法执行外部属性绑定逻辑
	/********************【主线，重点关注】********************/
	getBinder().bind(annotation.prefix(), target, bindHandler);
}
```
上面代码的主要逻辑是：
1. 先获取`target`对象（对应`XxxProperties`类）上的`@ConfigurationProperties`注解和校验器（若有）;
2. 然后再根据获取的的`@ConfigurationProperties`注解和校验器来获得`BindHandler`对象，`BindHandler`的作用是用于在属性绑定时来处理一些附件逻辑;在<font color=blue>8.1节分析</font>.
3. 最后再获取一个`Binder`对象，调用其`bind`方法来执行外部属性绑定的逻辑,在<font color=blue>8.2节分析</font>.

# 8.1 获取BindHandler对象以便在属性绑定时来处理一些附件逻辑
我们在看`getBindHandler`方法的逻辑前先来认识下`BindHandler`是干啥的。

`BindHandler`是一个父类接口，用于在属性绑定时来处理一些附件逻辑。我们先看下`BindHandler`的类图，好有一个整体的认识：


![](https://user-gold-cdn.xitu.io/2020/3/12/170cd15db2ca30b5?w=1015&h=259&f=png&s=16614)
<center>图1</center>

可以看到`AbstractBindHandler`作为抽象基类实现了`BindHandler`接口，其又有四个具体的子类分别是`IgnoreTopLevelConverterNotFoundBindHandler`,`NoUnboundElementsBindHandler`,`IgnoreErrorsBindHandler`和`ValidationBindHandler`。

1. `IgnoreTopLevelConverterNotFoundBindHandler`：在处理外部属性绑定时的默认`BindHandler`，当属性绑定失败时会忽略最顶层的`ConverterNotFoundException`；
2. `NoUnboundElementsBindHandler`：用来处理配置文件配置的未知的属性；
3. `IgnoreErrorsBindHandler`：用来忽略无效的配置属性例如类型错误；
4. `ValidationBindHandler`：利用校验器对绑定的结果值进行校验。

分析完类关系后，我们再来看下`BindHandler`接口提供了哪些方法在外部属性绑定时提供一些额外的附件逻辑，直接看代码：
```java
// BindHandler.java

public interface BindHandler {

	/**
	 * Default no-op bind handler.
	 */
	BindHandler DEFAULT = new BindHandler() {

	};

	// onStart方法在外部属性绑定前被调用
	default <T> Bindable<T> onStart(ConfigurationPropertyName name, Bindable<T> target,
			BindContext context) {
		return target;
	}

	// onSuccess方法在外部属性成功绑定时被调用，该方法能够改变最终返回的属性值或对属性值进行校验
	default Object onSuccess(ConfigurationPropertyName name, Bindable<?> target,
			BindContext context, Object result) {
		return result;
	}

	// onFailure方法在外部属性绑定失败（包括onSuccess方法里的逻辑执行失败）时被调用，
	// 该方法可以用来catch住相关异常或者返回一个替代的结果（跟微服务的降级结果有点类似，嘿嘿）
	default Object onFailure(ConfigurationPropertyName name, Bindable<?> target,
			BindContext context, Exception error) throws Exception {
		throw error;
	}

	// 当外部属性绑定结束时（不管绑定成功还是失败）被调用
	default void onFinish(ConfigurationPropertyName name, Bindable<?> target,
			BindContext context, Object result) throws Exception {
	}
}

```
可以看到`BindHandler`接口定义了`onStart`,`onSuccess`,`onFailure`和`onFinish`方法，这四个方法分别会在执行外部属性绑定时的不同时机会被调用，在属性绑定时用来添加一些额外的处理逻辑，比如在`onSuccess`方法改变最终绑定的属性值或对属性值进行校验，在`onFailure`方法`catch`住相关异常或者返回一个替代的绑定的属性值。

知道了`BindHandler`是在属性绑定时添加一些额外的附件处理逻辑后，我们再来看下`getBindHandler`方法的逻辑，直接上代码：
```java
// ConfigurationPropertiesBinder.java

// 注意BindHandler的设计技巧，应该是责任链模式，非常巧妙，值得借鉴
private BindHandler getBindHandler(ConfigurationProperties annotation,
		List<Validator> validators) {
	// 新建一个IgnoreTopLevelConverterNotFoundBindHandler对象，这是个默认的BindHandler对象
	BindHandler handler = new IgnoreTopLevelConverterNotFoundBindHandler();
	// 若注解@ConfigurationProperties的ignoreInvalidFields属性设置为true，
	// 则说明可以忽略无效的配置属性例如类型错误，此时新建一个IgnoreErrorsBindHandler对象
	if (annotation.ignoreInvalidFields()) {
		handler = new IgnoreErrorsBindHandler(handler);
	}
	// 若注解@ConfigurationProperties的ignoreUnknownFields属性设置为true，
	// 则说明配置文件配置了一些未知的属性配置，此时新建一个ignoreUnknownFields对象
	if (!annotation.ignoreUnknownFields()) {
		UnboundElementsSourceFilter filter = new UnboundElementsSourceFilter();
		handler = new NoUnboundElementsBindHandler(handler, filter);
	}
	// 如果@Valid注解不为空，则创建一个ValidationBindHandler对象
	if (!validators.isEmpty()) {
		handler = new ValidationBindHandler(handler,
				validators.toArray(new Validator[0]));
	}
	// 遍历获取的ConfigurationPropertiesBindHandlerAdvisor集合，
	// ConfigurationPropertiesBindHandlerAdvisor目前只在测试类中有用到
	for (ConfigurationPropertiesBindHandlerAdvisor advisor : getBindHandlerAdvisors()) {
		// 对handler进一步处理
		handler = advisor.apply(handler);
	}
	// 返回handler
	return handler;
}
```
`getBindHandler`方法的逻辑很简单，主要是根据传入的`@ConfigurationProperties`注解和`validators`校验器来创建不同的`BindHandler`具体实现类：
1. 首先`new`一个`IgnoreTopLevelConverterNotFoundBindHandler`作为默认的`BindHandler`;
2. 若`@ConfigurationProperties`注解的属性`ignoreInvalidFields`值为`true`，那么再`new`一个`IgnoreErrorsBindHandler`对象，把刚才新建的`IgnoreTopLevelConverterNotFoundBindHandler`对象作为构造参数传入赋值给`AbstractBindHandler`父类的`parent`属性；
3. 若`@ConfigurationProperties`注解的属性`ignoreUnknownFields`值为`false`，那么再`new`一个`UnboundElementsSourceFilter`对象，把之前构造的`BindHandler`对象作为构造参数传入赋值给`AbstractBindHandler`父类的`parent`属性；
4. ......以此类推，前一个`handler`对象作为后一个`hangdler`对象的构造参数，就这样利用`AbstractBindHandler`父类的`parent`属性将每一个`handler`链起来，最后再得到最终构造的`handler`。

> **GET技巧**：上面的这个设计模式是不是很熟悉，这个就是**责任链模式**。我们学习源码，同时也是学习别人怎么熟练运用设计模式。责任链模式的应用案例有很多，比如`Dubbo`的各种`Filter`们（比如`AccessLogFilter`是用来记录服务的访问日志的，`ExceptionFilter`是用来处理异常的...），我们一开始学习java web时的`Servlet`的`Filter`,`MyBatis`的`Plugin`们以及`Netty`的`Pipeline`都采用了责任链模式。

我们了解了`BindHandler`的作用后，再来紧跟主线，看属性绑定是如何绑定的？

# 8.2 获取Binder对象用于进行属性绑定【主线】
这里接<font color=blue>8 ConfigurationPropertiesBinder</font>节代码中标注`【4】`的主线代码`getBinder().bind(annotation.prefix(), target, bindHandler);`.

可以看到这句代码主要做了两件事：
1. 调用`getBinder`方法获取用于属性绑定的`Binder`对象；
2. 调用`Binder`对象的`bind`方法进行外部属性绑定到`@ConfigurationProperties`注解的`XxxProperties`类的属性上。

那么我们先看下`getBinder`方法源码：
```java
// ConfigurationPropertiesBinder.java

private Binder getBinder() {
	// Binder是一个能绑定ConfigurationPropertySource的容器对象
	if (this.binder == null) {
		// 新建一个Binder对象，这个binder对象封装了ConfigurationPropertySources，
		// PropertySourcesPlaceholdersResolver，ConversionService和PropertyEditorInitializer对象
		this.binder = new Binder(getConfigurationPropertySources(), // 将PropertySources对象封装成SpringConfigurationPropertySources对象并返回
				getPropertySourcesPlaceholdersResolver(), getConversionService(), // 将PropertySources对象封装成PropertySourcesPlaceholdersResolver对象并返回，从容器中获取到ConversionService对象
				getPropertyEditorInitializer()); // 得到Consumer<PropertyEditorRegistry>对象，这些初始化器用来配置property editors，property editors通常可以用来转换值
	}
	// 返回binder
	return this.binder;
}
```
可以看到`Binder`对象封装了`ConfigurationPropertySources`,`PropertySourcesPlaceholdersResolver`,`ConversionService`和`PropertyEditorInitializer`这四个对象，`Binder`对象封装了这四个哥们肯定是在后面属性绑定逻辑中会用到，先看下这四个对象是干嘛的：
* `ConfigurationPropertySources`:外部配置文件的属性源，由`ConfigFileApplicationListener`监听器负责触发读取；
* `PropertySourcesPlaceholdersResolver`:解析属性源中的占位符`${}`；
* `ConversionService`:对属性类型进行转换
* `PropertyEditorInitializer`:用来配置`property editors`


那么，我们获取了`Binder`属性绑定器后，再来看下它的`bind`方法是如何执行属性绑定的。
```java
// Binder.java

public <T> BindResult<T> bind(String name, Bindable<T> target, BindHandler handler) {
	// ConfigurationPropertyName.of(name)：将name（这里指属性前缀名）封装到ConfigurationPropertyName对象中
	// 将外部配置属性绑定到目标对象target中
	return bind(ConfigurationPropertyName.of(name), target, handler); 
}

public <T> BindResult<T> bind(ConfigurationPropertyName name, Bindable<T> target,
		BindHandler handler) {
	Assert.notNull(name, "Name must not be null");
	Assert.notNull(target, "Target must not be null");
	handler = (handler != null) ? handler : BindHandler.DEFAULT;
	// Context是Binder的内部类，实现了BindContext，Context可以理解为Binder的上下文，可以用来获取binder的属性比如Binder的sources属性
	Context context = new Context();
	// 进行属性绑定，并返回绑定属性后的对象bound，注意bound的对象类型是T，T就是@ConfigurationProperties注解的类比如ServerProperties
	/********【主线，重点关注】************/
	T bound = bind(name, target, handler, context, false);
	// 将刚才返回的bound对象封装到BindResult对象中并返回
	return BindResult.of(bound);
}
```
上面代码中首先创建了一个`Context`对象，`Context`是`Binder`的内部类，为`Binder`的上下文，利用`Context`上下文可以获取`Binder`的属性比如获取`Binder`的`sources`属性值并绑定到`XxxProperties`属性中。然后我们再紧跟主线看下` bind(name, target, handler, context, false)`方法源码：
```java
// Binder.java

protected final <T> T bind(ConfigurationPropertyName name, Bindable<T> target,
		BindHandler handler, Context context, boolean allowRecursiveBinding) {
	// 清空Binder的configurationProperty属性值
	context.clearConfigurationProperty(); 
	try { 
		// 【1】调用BindHandler的onStart方法，执行一系列的责任链对象的该方法
		target = handler.onStart(name, target, context);
		if (target == null) {
			return null;
		}// 【2】调用bindObject方法对Bindable对象target的属性进行绑定外部配置的值，并返回赋值给bound对象。
		// 举个栗子：比如设置了server.port=8888,那么该方法最终会调用Binder.bindProperty方法，最终返回的bound的value值为8888
		/************【主线：重点关注】***********/
		Object bound = bindObject(name, target, handler, context, 
				allowRecursiveBinding);
		// 【3】封装handleBindResult对象并返回，注意在handleBindResult的构造函数中会调用BindHandler的onSucess，onFinish方法
		return handleBindResult(name, target, handler, context, bound); 
	}
	catch (Exception ex) {
		return handleBindError(name, target, handler, context, ex);
	}
}
```
上面代码的注释已经非常详细，这里不再详述。我们接着紧跟主线来看看`bindObject`方法源码:
```java
// Binder.java

private <T> Object bindObject(ConfigurationPropertyName name, Bindable<T> target,
		BindHandler handler, Context context, boolean allowRecursiveBinding) {
	// 从propertySource中的配置属性，获取ConfigurationProperty对象property即application.properties配置文件中若有相关的配置的话，
	// 那么property将不会为null。举个栗子：假如你在配置文件中配置了spring.profiles.active=dev，那么相应property值为dev；否则为null
	ConfigurationProperty property = findProperty(name, context);
	// 若property为null，则不会执行后续的属性绑定相关逻辑
	if (property == null && containsNoDescendantOf(context.getSources(), name)) {
		// 如果property == null，则返回null
		return null;
	}
	// 根据target类型获取不同的Binder，可以是null（普通的类型一般是Null）,MapBinder,CollectionBinder或ArrayBinder
	AggregateBinder<?> aggregateBinder = getAggregateBinder(target, context);
	// 若aggregateBinder不为null比如配置了spring.profiles属性（当然包括其子属性比如spring.profiles.active等）
	if (aggregateBinder != null) {
		// 若aggregateBinder不为null，则调用bindAggregate并返回绑定后的对象
		return bindAggregate(name, target, handler, context, aggregateBinder);
	}
	// 若property不为null
	if (property != null) {
		try {
			// 绑定属性到对象中，比如配置文件中设置了server.port=8888，那么将会最终调用bindProperty方法进行属性设置
			return bindProperty(target, context, property);
		}
		catch (ConverterNotFoundException ex) {
			// We might still be able to bind it as a bean
			Object bean = bindBean(name, target, handler, context,
					allowRecursiveBinding);
			if (bean != null) {
				return bean;
			}
			throw ex;
		}
	}
	// 只有@ConfigurationProperties注解的类进行外部属性绑定才会走这里
	/***********************【主线，重点关注】****************************/
	return bindBean(name, target, handler, context, allowRecursiveBinding);
}
```
由上代码中可以看到`bindObject`中执行属性绑定的逻辑会根据不同的属性类型进入不同的绑定逻辑中，举个栗子：
1. `application.properties`配置文件中配置了`spring.profiles.active=dev`的话，那么将会进入`return bindAggregate(name, target, handler, context, aggregateBinder);`这个属性绑定的代码逻辑；
2. `application.properties`配置文件中配置了`server.port=8081`的话，那么将会进入`return bindBean(name, target, handler, context, allowRecursiveBinding);`的属性绑定的逻辑。

因此我们再次紧跟主线，进入`@ConfigurationProperties`注解的`XxxProperties`类的属性绑定逻辑中的`bindBean`方法中：
```java
// Binder.java

private Object bindBean(ConfigurationPropertyName name, Bindable<?> target, // name指的是ConfigurationProperties的前缀名
		BindHandler handler, Context context, boolean allowRecursiveBinding) {
	// 这里做一些ConfigurationPropertyState的相关检查
	if (containsNoDescendantOf(context.getSources(), name)
			|| isUnbindableBean(name, target, context)) {
		return null;
	}// 这里新建一个BeanPropertyBinder的实现类对象，注意这个对象实现了bindProperty方法
	BeanPropertyBinder propertyBinder = (propertyName, propertyTarget) -> bind(
			name.append(propertyName), propertyTarget, handler, context, false);
	/**
	 * (propertyName, propertyTarget) -> bind(
	 * 				name.append(propertyName), propertyTarget, handler, context, false);
	 * 	等价于
	 * 	new BeanPropertyBinder() {
	 *		Object bindProperty(String propertyName, Bindable<?> target){
	 *			bind(name.append(propertyName), propertyTarget, handler, context, false);
	 *		}
	 * 	}
	 */
	// type类型即@ConfigurationProperties注解标注的XxxProperties类
	Class<?> type = target.getType().resolve(Object.class);
	if (!allowRecursiveBinding && context.hasBoundBean(type)) {
		return null;
	}
	// 这里应用了java8的lambda语法，作为没怎么学习java8的lambda语法的我，不怎么好理解下面的逻辑，哈哈
	// 真正实现将外部配置属性绑定到@ConfigurationProperties注解的XxxProperties类的属性中的逻辑应该就是在这句lambda代码了
	/*******************【主线】***************************/
	return context.withBean(type, () -> {
		Stream<?> boundBeans = BEAN_BINDERS.stream()
				.map((b) -> b.bind(name, target, context, propertyBinder));
		return boundBeans.filter(Objects::nonNull).findFirst().orElse(null);
	});
	// 根据上面的lambda语句翻译如下：
	/** 这里的T指的是各种属性绑定对象，比如ServerProperties
	 * return context.withBean(type, new Supplier<T>() {
	 * 	T get() {
	 * 		Stream<?> boundBeans = BEAN_BINDERS.stream()
	 * 					.map((b) -> b.bind(name, target, context, propertyBinder));
	 * 			return boundBeans.filter(Objects::nonNull).findFirst().orElse(null);
	 *        }
	 *  });
	 */
}
```

从上面代码中，我们追根究底来到了外部配置属性绑定到`XxxProperties`类属性中的比较底层的代码了，可以看到属性绑定的逻辑应该就在上面代码标注`【主线】`的`lambda`代码处了。这里就不再详述了，因为这个属于SpringBoot的属性绑定`Binder`的范畴，`Binder`相关类是SpringBoot2.0才出现的，即对之前的属性绑定相关代码进行推翻重写了。属性绑定相关的源码也比较多，后续有需要再另开一篇来分析探究吧。


# 9 小结

好了，外部配置属性值是如何被绑定到`XxxProperties`类属性上的源码分析就到此结束了，又是蛮长的一篇文章，不知自己表述清楚没，重要步骤现总结下：
1. 首先是`@EnableConfigurationProperties`注解`import`了`EnableConfigurationPropertiesImportSelector`后置处理器；
2. `EnableConfigurationPropertiesImportSelector`后置处理器又向`Spring`容器中注册了`ConfigurationPropertiesBeanRegistrar`和`ConfigurationPropertiesBindingPostProcessorRegistrar`这两个`bean`；
3. 其中`ConfigurationPropertiesBeanRegistrar`向`Spring`容器中注册了`XxxProperties`类型的`bean`；`ConfigurationPropertiesBindingPostProcessorRegistrar`向`Spring`容器中注册了`ConfigurationBeanFactoryMetadata`和`ConfigurationPropertiesBindingPostProcessor`两个后置处理器；
4. `ConfigurationBeanFactoryMetadata`后置处理器在初始化`bean` `factory`时将`@Bean`注解的元数据存储起来，以便在后续的外部配置属性绑定的相关逻辑中使用；
5. `ConfigurationPropertiesBindingPostProcessor`后置处理器将外部配置属性值绑定到`XxxProperties`类属性的逻辑委托给`ConfigurationPropertiesBinder`对象，然后`ConfigurationPropertiesBinder`对象又最终将属性绑定的逻辑委托给`Binder`对象来完成。

可见，重要的是上面的**第5步**。

**原创不易，帮忙点个赞呗！**

**PS**：本来打算这篇开始分析SpringBoot的启动流程的，但是回过头去看看自动配置的相关源码，还有蛮多没有分析的，因此再来一波自动配置相关的源码先。

由于笔者水平有限，若文中有错误还请指出，谢谢。


参考：
1,[JSR-303](https://www.jianshu.com/p/554533f88370)

---------------------------------------------------
欢迎关注【源码笔记】公众号，一起学习交流。

<img src="https://user-gold-cdn.xitu.io/2020/3/13/170d433d335f79e2?w=160&h=166&f=png&s=46879" width = "100" height = "100" align=left />

