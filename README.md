【**源码笔记**】专注于Java后端系列框架的源码分析。若觉得源码分析文章不错，欢迎Star哦。

由于【源码笔记】今年2月初才开始写源码分析文章，因此目前源码分析文章还不是很多，计划每周持续推出一到两篇Java后端框架源码系列的文章，随着时间的积累，Java后端源码分析文章肯定会越来越多，越来越丰富哦，敬请关注。

### 目录

===================**源码阅读感悟&&阅读技巧**======================
1. [跟大家聊聊我们为什么要学习源码？学习源码对我们有用吗？](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/%E8%B7%9F%E5%A4%A7%E5%AE%B6%E8%81%8A%E8%81%8A%E6%88%91%E4%BB%AC%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E5%AD%A6%E4%B9%A0%E6%BA%90%E7%A0%81%EF%BC%9F%E5%AD%A6%E4%B9%A0%E6%BA%90%E7%A0%81%E5%AF%B9%E6%88%91%E4%BB%AC%E6%9C%89%E7%94%A8%E5%90%97%EF%BC%9F.md)
2. [分析开源项目源码，我们该如何入手分析？（授人以渔）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/%E5%88%86%E6%9E%90%E5%BC%80%E6%BA%90%E9%A1%B9%E7%9B%AE%E6%BA%90%E7%A0%81%EF%BC%8C%E6%88%91%E4%BB%AC%E8%AF%A5%E5%A6%82%E4%BD%95%E5%85%A5%E6%89%8B%E5%88%86%E6%9E%90%EF%BC%9F%EF%BC%88%E6%8E%88%E4%BA%BA%E4%BB%A5%E6%B8%94%EF%BC%89.md)

================**JUC源码专题持续更新中...**====================
1. [Java是如何实现Future模式的？万字详解！](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/JUC/Java%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0Future%E6%A8%A1%E5%BC%8F%E7%9A%84%EF%BC%9F%E4%B8%87%E5%AD%97%E8%AF%A6%E8%A7%A3%EF%BC%81.md)
2. 持续更新中...
* JUC源码分析专题：https://github.com/yuanmabiji/Java-SourceCode-Blogs/tree/master/JUC
* JUC源码解析项目（带中文注释）：https://github.com/yuanmabiji/jdk1.8-sourcecode-blogs

================**SpringBoot源码专题持续更新中...**====================
1. [如何搭建自己的SpringBoot源码调试环境？ SpringBoot源码（一）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/1%20%E5%A6%82%E4%BD%95%E6%90%AD%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84SpringBoot%E6%BA%90%E7%A0%81%E8%B0%83%E8%AF%95%E7%8E%AF%E5%A2%83%EF%BC%9F%20%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E4%B8%80%EF%BC%89.md)
2. [如何分析SpringBoot源码模块及结构？ SpringBoot源码（二）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/2%20%E5%A6%82%E4%BD%95%E5%88%86%E6%9E%90SpringBoot%E6%BA%90%E7%A0%81%E6%A8%A1%E5%9D%97%E5%8F%8A%E7%BB%93%E6%9E%84%EF%BC%9F%20%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E4%BA%8C%EF%BC%89.md)
3. [助力SpringBoot自动配置的条件注解原理揭秘 SpringBoot源码（三）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/3%20%E5%8A%A9%E5%8A%9BSpringBoot%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE%E7%9A%84%E6%9D%A1%E4%BB%B6%E6%B3%A8%E8%A7%A3%E5%8E%9F%E7%90%86%E6%8F%AD%E7%A7%98%20%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E4%B8%89%EF%BC%89.md)
4. [SpringBoot是如何实现自动配置的？ SpringBoot源码（四）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/4%20SpringBoot%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE%E7%9A%84%EF%BC%9F%20%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E5%9B%9B%EF%BC%89.md)
5. [SpringBoot的配置属性值是如何绑定的？ SpringBoot源码（五）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/5%20SpringBoot%E7%9A%84%E9%85%8D%E7%BD%AE%E5%B1%9E%E6%80%A7%E5%80%BC%E6%98%AF%E5%A6%82%E4%BD%95%E7%BB%91%E5%AE%9A%E7%9A%84%EF%BC%9F%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E4%BA%94%EF%BC%89.md)
6. [SpringBoot内置的各种Starter是怎样构建的？ SpringBoot源码（六）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/6%20SpringBoot%E5%86%85%E7%BD%AE%E7%9A%84%E5%90%84%E7%A7%8DStarter%E6%98%AF%E6%80%8E%E6%A0%B7%E6%9E%84%E5%BB%BA%E7%9A%84%EF%BC%9F%20%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E5%85%AD%EF%BC%89.md)
7. [SpringBoot的启动流程是怎样的？SpringBoot源码（七）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/7%20SpringBoot%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84%EF%BC%9FSpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E4%B8%83%EF%BC%89.md)
8. [SpringApplication对象是如何构建的？ SpringBoot源码（八）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/8%20SpringApplication%E5%AF%B9%E8%B1%A1%E6%98%AF%E5%A6%82%E4%BD%95%E6%9E%84%E5%BB%BA%E7%9A%84%EF%BC%9F%20SpringBoot%E6%BA%90%E7%A0%81%EF%BC%88%E5%85%AB%EF%BC%89.md)
9. [SpringBoot事件监听机制源码分析(上) SpringBoot源码(九)](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/9%20SpringBoot%E4%BA%8B%E4%BB%B6%E7%9B%91%E5%90%AC%E6%9C%BA%E5%88%B6%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(%E4%B8%8A)%20SpringBoot%E6%BA%90%E7%A0%81(%E4%B9%9D).md)
10. [SpringBoot内置生命周期事件详解  SpringBoot源码(十)](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/SpringBoot/10%20SpringBoot%E5%86%85%E7%BD%AE%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E4%BA%8B%E4%BB%B6%E8%AF%A6%E8%A7%A3%20%20SpringBoot%E6%BA%90%E7%A0%81(%E5%8D%81).md)
11. 持续更新中...
* SpringBoot源码分析专题：https://github.com/yuanmabiji/Java-SourceCode-Blogs/tree/master/SpringBoot
* SpringBoot源码解析项目（带中文注释）：https://github.com/yuanmabiji/spring-boot-2.1.0.RELEASE


================**Spring5源码专题持续更新中...**====================
1. [模仿Spring事件机制实现自定义事件驱动编程 Spring源码（一）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/Spring/1%20%E6%A8%A1%E4%BB%BFSpring%E4%BA%8B%E4%BB%B6%E6%9C%BA%E5%88%B6%E5%AE%9E%E7%8E%B0%E8%87%AA%E5%AE%9A%E4%B9%89%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E7%BC%96%E7%A8%8B%20Spring%E6%BA%90%E7%A0%81%EF%BC%88%E4%B8%80%EF%BC%89.md)
2. [Spring是如何实现事件监听机制的？ Spring源码（二）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/Spring/2%20Spring%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E4%BA%8B%E4%BB%B6%E7%9B%91%E5%90%AC%E6%9C%BA%E5%88%B6%E7%9A%84%EF%BC%9F%20%20Spring%E6%BA%90%E7%A0%81%EF%BC%88%E4%BA%8C%EF%BC%89.md)
3. 持续更新中...
* Spring5源码分析专题：https://github.com/yuanmabiji/Java-SourceCode-Blogs/tree/master/Spring
* Spring5源码解析项目（带中文注释）：待提供


================**JDK源码专题持续更新中...**====================
1. [Java是如何实现自己的SPI机制的？ JDK源码（一）](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/JDK/1%20Java%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E8%87%AA%E5%B7%B1%E7%9A%84SPI%E6%9C%BA%E5%88%B6%E7%9A%84%EF%BC%9F%20JDK%E6%BA%90%E7%A0%81%EF%BC%88%E4%B8%80%EF%BC%89.md)
2. 持续更新中...
* JDK源码分析专题：https://github.com/yuanmabiji/Java-SourceCode-Blogs/tree/master/JDK
* JDK源码解析项目（带中文注释）：https://github.com/yuanmabiji/jdk1.8-sourcecode-blogs

================**Disruptor源码专题持续更新中...**====================
1. [初识Disruptor框架！](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/Disruptor/初识Disruptor框架.md)
2. 持续更新中...
* Disruptor源码分析专题：https://github.com/yuanmabiji/Java-SourceCode-Blogs/tree/master/Disruptor
* Disruptor源码解析项目（带中文注释）：https://github.com/yuanmabiji/disruptor

================**TODO LIST**====================

* SpringMVC
* Mybatis
* Dubbo
* Netty
* RocketMQ
* SpringCloud
* Shiro
* Tomcat
* Seata
* JUC
* Zookeeper
* .....

