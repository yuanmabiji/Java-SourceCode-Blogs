
## 1 前言
这是SpringBoot2.1源码分析专题的第一篇文章，主要讲如何来搭建我们的源码阅读调试环境。如果有经验的小伙伴们可以略过此篇文章。
## 2 环境安装要求
* IntelliJ IDEA
* JDK1.8
* Maven3.5以上

## 3 从github上将SpringBoot源码项目下载下来
首先提供**SpringBoot2.1.0**的github地址：
https://github.com/spring-projects/spring-boot/tree/v2.1.0.RELEASE

因为要进行阅读源码和分析源码项目，我们是不是要在里面写一些注释帮助我们阅读理解源码，因此需要将SpringBoot源码项目fork到自己的github仓库中，然后再利用**git clone url**命令将已经fork到自己github仓库的SpringBoot源码拉取下来即可。
但由于以上方式往往很慢，通常会超时，所以笔者直接将SpringBoot项目直接下载下来，然后再导入IDEA中。

![](https://user-gold-cdn.xitu.io/2020/2/23/1706fb164630bfac?w=1310&h=484&f=png&s=70931)
## 4 将SpringBoot源码项目导入到IDEA中
将刚才下载的spring-boot2.1.0.RELEASE项目选择maven方式导入到IDEA中，然后一直next即可导入完成，注意选择JDK版本是1.8，maven版本是3.5+。

![](https://user-gold-cdn.xitu.io/2020/2/23/1706fb50079899ac?w=734&h=266&f=png&s=12373)
此时下载maven依赖是一个漫长的等待过程，建议maven没有配置（阿-里-云）仓库的小伙伴们配置一下，这样下载速度会快很多。参考[配置maven使用（阿-里-云）仓库](https://blog.csdn.net/zhuzj12345/article/details/93200211)进行配置即可。
## 5 编译构建SpringBoot源码项目
此时导入项目后，我们进行编译构建SpringBoot源码项目了，在构建之前做两个配置：
1. 我们要禁用maven的代码检查，在根pom.xml中增加一下配置即可，如下图：
> ![](https://user-gold-cdn.xitu.io/2020/2/23/1706fbc5725ffaf8?w=947&h=303&f=png&s=33333)
2. 可能有的小伙伴们的pom.xml文件的project标签上显示`java.lang.OutOfMemoryError`错误，这是因为IDEA里的Maven的importer设置的JVM最大堆内存过小而导致的，如下图,此时可参考[Maven依赖包导入错误（IntelliJ IDEA）](https://blog.csdn.net/w605283073/article/details/85107497)解决即可。
> ![](https://user-gold-cdn.xitu.io/2020/2/23/1706fc20d5eaba6b?w=688&h=254&f=png&s=98291)

进行了上面的两点配置后，此时我们就可以直接执行以下maven命令来编译构建源码项目了。
```
mvn clean install -DskipTests -Pfast
```

![](https://user-gold-cdn.xitu.io/2020/2/23/1706fc5696712841?w=697&h=92&f=png&s=8216)
此时又是漫长的等待，我这里等待5分钟左右就显示构建成功了，如下图：
![](https://user-gold-cdn.xitu.io/2020/2/23/1706fdcd5edaceb9?w=890&h=563&f=png&s=56801)
## 6 运行SpringBoot自带的sample
因为SpringBoot源码中的spring-boot-samples模块自带了很多DEMO样例，我们可以利用其中的一个sample来测试运行刚刚构建的springboot源码项目即可。但此时发现spring-boot-samples模块是灰色的，如下图：
![](https://user-gold-cdn.xitu.io/2020/2/23/1706fca29948aa2a?w=300&h=136&f=png&s=5880)
这是因为spring-boot-samples模块没有被添加到根pom.xml中，此时将其添加到根pom.xml中即可，增加如下配置，如下图：
![](https://user-gold-cdn.xitu.io/2020/2/23/1706fcb839241e47?w=815&h=330&f=png&s=34161)
此时我们挑选spring-boot-samples模块下的spring-boot-sample-tomcat样例项目来测试好了，此时启动`SampleTomcatApplication`的`main`函数，启动成功界面如下：

![](https://user-gold-cdn.xitu.io/2020/2/23/1706fde9a5073f65?w=1843&h=444&f=png&s=130677)
然后我们再在浏览器发送一个HTTP请求，此时可以看到服务端成功返回响应，说明此时SpringBoot源码环境就已经构建成功了，接下来我们就可以进行调试了，如下图：

![](https://user-gold-cdn.xitu.io/2020/2/23/1706fd98c2621c7b?w=411&h=161&f=png&s=9614)

## 7 动手实践环节
前面已经成功构建了SpringBoot的源码阅读环境，小伙伴们记得自己动手搭建一套属于自己的SpringBoot源码调试环境哦，阅读源码动手调试很重要，嘿嘿。

**下节预告**：
<font color=Blue>我们该如何去分析SpringBoot源码涉及模块及结构？--SpringBoot源码（二）</font>


**原创不易，帮忙Star一下呗**！

注：该源码分析对应SpringBoot版本为**2.1.0.RELEASE**，本文对应的SpringBoot源码解析项目github地址：https://github.com/yuanmabiji/spring-boot-2.1.0.RELEASE

