注：该源码分析对应SpringBoot版本为**2.1.0.RELEASE**
## 1 前言
本篇接
[如何搭建自己的SpringBoot源码调试环境？--SpringBoot源码（一）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/1%20%E5%A6%82%E4%BD%95%E6%90%AD%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84SpringBoot%E6%BA%90%E7%A0%81%E8%B0%83%E8%AF%95%E7%8E%AF%E5%A2%83%EF%BC%9F%20%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E4%B8%80%EF%BC%89.md)。

前面搭建好了自己本地的SpringBoot源码调试环境后，此时我们不要急着下手进入到具体的源码调试细节中，**刚开始阅读源码，此时我们一定要对项目结构等有一个整体的认识，然后再进行源码分析调试**。推荐阅读下笔者之前写的的[分析开源项目源码，我们该如何入手分析？](https://juejin.im/post/5e4d06b451882549670673c5)一文，干货满满哦。

## 2 SpringBoot源码模块一览
我们先来对SpringBoot的源码模块来一个大致的了解，如下图：

![](https://user-gold-cdn.xitu.io/2020/2/23/1707109ab6ff4c1f?w=431&h=273&f=png&s=17190)
从上图可以看到，主要有以下四个模块：
* **spring-boot-project**：整个SpringBoot框架全部功能在这个模块实现，SpringBoot项目95%的代码都在这里实现，源码总共有25万行左右。
* **Spring-boot-samples**：这个是SpringBoot给小伙伴们赠送的福利，里面包含了各种各样使用SpringBoot的简单demo，我们调试阅读源码的时候可以充分利用该模块。
* **Spring-boot-sample-invoker**：这个模块应该是跟sample模块有关，注意根pom.xml中有这么一句话：`Samples are built via the invoker plugin`，该模块无代码。
* **Spring-boot-tests**：这个模块SpringBoot的测试模块，跟部署测试和集成测试有关。

因为SpringBoot的全部功能在spring-boot-project模块实现，因此下面重点来介绍下 spring-boot-project 模块。
## 3 spring-boot-project源码模块详解
先来看下spring-boot-project整体模块结构，如下图，然后我们再逐个来介绍：

![](https://user-gold-cdn.xitu.io/2020/2/23/170711f4a6a14a10?w=300&h=347&f=png&s=13577)
#### 1) spring-boot-parent
这个模块没有代码，是spring-boot模块的父项目，被其他子模块继承。
#### 2) spring-boot
这个模块是SpringBoot项目的核心，可以说一些基础核心的功能都在这里实现，为SpringBoot的其他模块组件功能提供了支持，主要包括以下核心功能：
* `SpringApplication`类，这个是SpringBoot的启动类，提供了一个静态的`run`方法来启动程序，该类主要用来创建并且刷新Spring容器`ApplicationContext`.
* 支持选择不同的容器比如Tomcat,Jetty等来作为应用的嵌入容器，这个是SpringBoot的新特性之一。
* 外部配置支持，这个指的是我们执行`java -jar xxx.jar`命令时可以带一些参数，比如执行`java -jar demo.jar --server.port=8888`来将应用端口修改为8888.
* 该模块内置了一些SpringBoot启动时的生命周期事件和一些容器初始化器(`ApplicationContext` initializers)，来执行一些SpringBoot启动时的初始化逻辑。

#### 3) spring-boot-autoconfigure
这个模块跟SpringBoot的自动配置有关，也是SpringBoot的新特性之一。比如SpringBoot能基于类路径来自动配置某个项目模块，自动配置最为关键的注解是`@EnableAutoConfiguration`,这个注解能触发Spring上下文的自动配置。另外一个重要的注解是`@Conditional`。
> 举个栗子，若`HSQLDB`在项目的类路径中，且我们没有配置任何其他数据库的连接，此时自动配置就会自动根据类路径来创建相应的`bean`。

除了根据类路径来进行自动配置外，还有根据容器中是否存在某个bean等方式来进行自动配置，这里不会进入到具体细节中。
#### 4) spring-boot-starters
这个模块是跟SpringBoot的起步依赖有关，也是SpringBoot的新特性之一。SpringBoot通过提供众多起步依赖降低项目依赖的复杂度。起步依赖其实就是利用maven项目模型将其他相关的依赖给聚合起来，里面各种依赖的版本号都给定义好，避免用户在引入依赖时出现各种版本冲突，方便了我们的使用。
> 举个栗子，我们要用到activemq时，此时可以直接引入`spring-boot-starter-activemq`起步依赖即可，若SpringBoot官网或第三方组织没有提供相应的SpringBoot起步依赖时，此时我们可以进行定制自己的起步依赖。

注意，该模块没有代码，主要是通过maven的pom.xml来组织各种依赖。


#### 5) spring-boot-cli
Spring Boot CLI是一个命令行工具，如果您想使用Spring快速开发，可以使用它。它允许您运行Groovy脚本，这意味着您有一个熟悉的类似Java的语法，而没有那么多样板代码。您还可以引导一个新项目或编写自己的命令。
#### 6) spring-boot-actuator
这个跟SpringBoot的监控有关，也是SpringBoot的新特性之一。可以通过HTTP端点或JMX等来管理和监控应用。审计、运行状况和度量收集可以自动应用到应用程序。这个监控模块是开箱即用的，提供了一系列端点包括`HealthEndpoint`, `EnvironmentEndpoint`和`BeansEndpoint`等端点。
#### 7) spring-boot-actuator-autoconfigure
这个模块为监控模块提供自动配置的功能，通常也是根据类路径来进行配置。比如`Micrometer`存在于类路径中，那么将会自动配置`MetricsEndpoint`。
#### 8) spring-boot-test
这个模式是spring-boot的跟测试有关的模块，包含了一些帮助我们测试的核心类和注解（比如`@SpringBootTest`）。

#### 9) spring-boot-dependencies
这个模块也没有代码，主要是定义了一些SpringBoot的maven相关的一些依赖及其版本。
#### 10) spring-boot-devtools
这个模块跟SpringBoot的热部署有关，即修改代码后无需重启应用即生效。
#### 11) spring-boot-docs
这个模块应该是跟文档相关的模块。
#### 12) spring-boot-properties-migrator
看到 migrator 这个单词，估计就是跟项目迁移有关，没有去细
究。
#### 13) spring-boot-test-autoconfigure
这个模块一看就是跟SpringBoot的测试的自动配置有关。
#### 14) spring-boot-tools
这个模块一看就是SpringBoot的工具相关的模块，提供了加载，maven插件,metadata和后置处理相关的支持。

上面介绍了这么多spring-boot模块下的子模块，不用慌，我们要进行解读的模块不多，我们真正要看的模块有`spring-boot`，`spring-boot-autoconfigure`，`spring-boot-starters`和`spring-boot-actuator`模块。

## 5 用一个思维导图来总结下SpringBoot源码项目的脉络

![](https://user-gold-cdn.xitu.io/2020/2/23/170728e220dc58a9?w=1247&h=749&f=png&s=281752)
## 6 SpringBoot模块之间的pom关系详解
前面弄清楚了SpringBoot的各个模块的具体功能，此时我们来看下SpringBoot模块的pom之间的关系是怎样的，因为项目是通过maven构建的，因此还是有必要去研究下这块关系滴。

先看SpringBoot源码项目的pom关系，如下图：

![](https://user-gold-cdn.xitu.io/2020/2/23/1707242f908d6dde?w=1122&h=565&f=png&s=51064)
根据上图可得出以下结论：
* `spring-boot-build(pom.xml)`是项目的根pom，其子pom有`spring-boot-project(pom.xml)`和`spring-boot-dependencies(pom.xml)`；
* `spring-boot-dependencies(pom.xml)`主要定义了SpringBoot项目的各种依赖及其版本，其子pom有`spring-boot-parent(pom.xml)`和`spring-boot-starter-parent(pom.xml)`；
* `spring-boot-project(pom.xml)`起到聚合module的作用，其子模块并不继承于它，而是继承于`spring-boot-parent(pom.xml)`；
* `spring-boot-parent(pom.xml)`是`spring-boot-project(pom.xml)`的子module，但继承的父pom为`spring-boot-dependencies(pom.xml)`，其定义了一些properties等相关的东西。其子pom为`spring-boot-project(pom.xml)`的子module（注意除去`spring-boot-dependencies(pom.xml)`），比如有`spring-boot(pom.xml)`,`spring-boot-starters(pom.xml)`和`spring-boot-actuator(pom.xml)`等；
* `spring-boot-starters(pom.xml)`是所有具体起步依赖的父pom，其子pom有`spring-boot-starter-data-jdbc(pom.xml)`和`spring-boot-starter-data-redis(pom.xml)`等。
* `spring-boot-starter-parent(pom.xml)`，是我们的所有具体SpringBoot项目的父pom，比如SpringBoot自带的样例的`spring-boot-samples(pom.xml)`是继承于它的。

SpringBoot的各模块之间的pom关系有点复杂，确实有点绕，如果看完上面的图片和解释还是不太清楚的话，建议小伙伴们自己打开idea的项目，逐个去捋一下。总之记得SpringBoot的一些父pom无非是做了一些版本管理，聚合模块之间的事情。

## 5 小结
好了，前面已经把SpringBoot源码项目的各个模块的功能和模块pom之间的关系给捋清楚了，总之刚开始分析项目源码，有一个整体的大局观很重要。

本来下节想先写SpringBoot的启动流程分析的，但由于之前研究过启动流程，所以就把启动流程分析放后点写了。下一节先对SpringBoot的新特性--自动配置的源码撸起来，因此下一节让我们先来揭开SpringBoot自动配置背后神秘的面纱吧，嘿嘿🤭。

**下节预告**：
<font color=Blue>SpringBoot自动配置的相关原理搞起来</font>


**原创不易，帮忙Star一下呗**！

参考：

1，https://github.com/spring-projects/spring-boot/tree/v2.1.0.RELEASE

2，https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#cli


