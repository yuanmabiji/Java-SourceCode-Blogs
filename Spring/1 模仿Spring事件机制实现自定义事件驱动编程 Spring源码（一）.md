![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)**注意：spring源码分析文章对应spring版本为 5.1.x**

# 1，概述

要想理解spring的事件机制，我觉得首先自己动手去撸一套简单的自定义事件驱动编程demo还是非常有必要滴，因为这样有助于理解spring事件机制。当然，这里也是模仿spring的事件机制的代码，不过下面看代码实现时可以先抛开spring的事件机制相关代码，将注意力集中到这个简单demo上即可。

在看这个自定义事件驱动编程时，首先要熟悉观察者设计模式，因为事件驱动编程可以说是观察者（发布订阅）模式的具体实现。推荐我的另一篇翻译的博文：[观察者模式--设计模式（一）](https://blog.csdn.net/biaolianlao0449/article/details/104246763)，有需要的小伙伴们可以先学习下哈。

下面正式开始手撸代码实现，首先先放上下面代码的github地址：

https://github.com/jinyue233/java-demo/tree/master/spring5-demo/src/main/java/com/jinyue/spring/event/mockspringevent

# 2，自定义事件驱动编程

因为这篇文章是spring事件机制的前置文章，因此这里自定义实现一个模拟容器（可以理解为spring容器，servltet容器等）的生命周期事件的简单demo。

## 2.1 事件

先来看一下事件的整体架构图，让大家先有对事件有一个整体的认识，如下图：

![img](https://user-gold-cdn.xitu.io/2020/2/11/1703314099225c59?w=637&h=332&f=png&s=24496)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**1，Event接口**

面向接口编程，首先先定义一个Event接口，该接口没有任何方法，可以说是事件的标志接口。

```
public interface Event extends Serializable {
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**2，AbstractContextEvent**

AbstractContextEvent是容器事件的基本抽象类，因为事件也可以携带数据，因此这里定义了一个timestamp属性，用来记录事件的发生时间。

```
public class AbstractContextEvent implements Event {
    private static final long serialVersionUID = -6159391039546783871L;

    private final long timestamp = System.currentTimeMillis();

    public final long getTimestamp() {
        return this.timestamp;
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**3，ContextStartEvent**

ContextStartEvent事件是AbstractContextEvent具体实现类，容器开始启动时触发，这里为了demo简单，这里不再定义任何事件逻辑，只是代表容器启动时的一个标志事件类。

```
public class ContextStartEvent extends AbstractContextEvent {
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



**4，ContextRunningEvent**



ContextRunningEvent事件是AbstractContextEvent具体实现类，容器启动后时触发，这里为了demo简单，这里不再定义任何事件逻辑，只是代表容器启动运行时时的一个标志事件类。

```
public class ContextRunningEvent extends AbstractContextEvent {
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**5，ContextDestroyEvent**



ContextDestroyEvent事件是AbstractContextEvent具体实现类，容器销毁时触发，这里为了demo简单，这里不再定义任何事件逻辑，只是代表容器销毁时的一个标志事件类。

```
public class ContextDestroyEvent extends AbstractContextEvent {
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 2.2 事件监听器

先来看一下事件监听器的整体架构图，让大家先有对事件监听器有一个整体的认识，如下图：

![img](https://user-gold-cdn.xitu.io/2020/2/11/1703314096a6a5e1?w=758&h=289&f=png&s=25133)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

其中EventListener是所有事件监听器的基类接口，，是事件监听器的标志类接口，被所有具体的事件监听器实现。然后ContextListener接口是容器事件监听器接口，继承了EventListener，主要定义了如下事件监听方法：

```
 void onApplicationEvent(T event);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

然后ContextListener接口被三个具体的容器生命周期事件监听器实现，分别是ContextStartEventListener（监听容器启动时的ContextStartEvent），ContextRunningEventListener（监听容器启动后运行时的ContextRunningEvent）和ContextDestroyEventListener（监听容器销毁时的ContextDestroyEvent）。

下面看具体的代码实现：

```
public interface EventListener {
}


public interface ContextListener<T extends AbstractContextEvent> extends EventListener {
    /**
     * Handle an application event.
     * @param event the event to respond to
     */
    void onApplicationEvent(T event);
}


public class ContextStartEventListener implements ContextListener<AbstractContextEvent> {
    /**
     * Handle an application event.
     *
     * @param event the event to respond to
     */
    public void onApplicationEvent(AbstractContextEvent event) {
        if (event instanceof ContextStartEvent) {
            System.out.println("容器启动。。。，启动时间为：" + event.getTimestamp());
        }
    }
}

public class ContextRunningEventListener implements ContextListener<AbstractContextEvent> {
    /**
     * Handle an application event.
     *
     * @param event the event to respond to
     */
    public void onApplicationEvent(AbstractContextEvent event) {
        if (event instanceof ContextRunningEvent) {
            System.out.println("容器开始运行。。。");
            try {
                Thread.sleep(3000);
                System.out.println("容器运行结束。。。");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class ContextDestroyEventListener implements ContextListener<AbstractContextEvent> {
    /**
     * Handle an application event.
     *
     * @param event the event to respond to
     */
    public void onApplicationEvent(AbstractContextEvent event) {
        if (event instanceof ContextDestroyEvent) {
            System.out.println("容器销毁。。。，销毁时间为：" + event.getTimestamp());
        }
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 2.3 事件发布器

先看下类图：

![img](https://user-gold-cdn.xitu.io/2020/2/11/1703314099136a60?w=538&h=63&f=png&s=7799)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

ApplicationEventMulticaster是发布事件的父类接口，主要定义了增加，删除，获取等操作事件监听器的的方法接口，此外，还定义了一个发布事件的方法。

SimpleApplicationEventMulticaster是ApplicationEventMulticaster事件发布器接口的默认实现类，主要承担发布事件的功能。其内部维护了一个事件监听器列表contextListeners，当发布事件时会遍历这些列表，然后再向每个监听器发布事件，通过设置async属性来决定同步广播事件还是异步广播事件。

下面看看实现代码：

```
public interface ApplicationEventMulticaster {
    void addContextListener(ContextListener<?> listener);

    void removeContextListener(ContextListener<?> listener);

    void removeAllListeners();

    void multicastEvent(AbstractContextEvent event);

}



public class SimpleApplicationEventMulticaster implements ApplicationEventMulticaster {
    // 是否异步发布事件
    private boolean async = false;
    // 线程池
    private Executor taskExecutor = new ThreadPoolExecutor(5, 5, 0, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
    // 事件监听器列表
    private List<ContextListener<?>> contextListeners = new ArrayList<ContextListener<?>>();

    
    public void addContextListener(ContextListener<?> listener) {
        contextListeners.add(listener);
    }

    public void removeContextListener(ContextListener<?> listener) {
        contextListeners.remove(listener);
    }

    public void removeAllListeners() {
        contextListeners.clear();
    }

    public void multicastEvent(AbstractContextEvent event) {
        doMulticastEvent(contextListeners, event);
    }

    private void doMulticastEvent(List<ContextListener<?>> contextListeners, AbstractContextEvent event) {
        for (ContextListener contextListener : contextListeners) {
            // 异步广播事件
            if (async) {
                taskExecutor.execute(() -> invokeListener(contextListener, event));
                // new Thread(() -> invokeListener(contextListener, event)).start();
            // 同步发布事件，阻塞的方式
            } else {
                invokeListener(contextListener, event);
            }
        }
    }

    private void invokeListener(ContextListener contextListener, AbstractContextEvent event) {
        contextListener.onApplicationEvent(event);
    }

    public void setAsync(boolean async) {
        this.async = async;
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 2.4 测试自定义的容器生命周期事件

那么直接上测试代码，下面只演示同步发布事件的功能：

```
public class MockSpringEventTest {

    @Test
    public void testContextLifecycleEventInSync() {
        // 新建SimpleApplicationEventMulticaster对象，并添加容器生命周期监听器
        ApplicationEventMulticaster eventMulticaster = new SimpleApplicationEventMulticaster();
        eventMulticaster.addContextListener(new ContextStartEventListener());
        eventMulticaster.addContextListener(new ContextRunningEventListener());
        eventMulticaster.addContextListener(new ContextDestroyEventListener());
        // 发射容器启动事件ContextStartEvent
        eventMulticaster.multicastEvent(new ContextStartEvent());
        // 发射容器正在运行事件ContextRunningEvent
        eventMulticaster.multicastEvent(new ContextRunningEvent());
        // 发射容器正在运行事件ContextDestroyEvent
        eventMulticaster.multicastEvent(new ContextDestroyEvent());
    }

    @Test
    public void testContextLifecycleEventInAsync() throws InterruptedException {
        // 新建SimpleApplicationEventMulticaster对象，并添加容器生命周期监听器
        ApplicationEventMulticaster eventMulticaster = new SimpleApplicationEventMulticaster();
        eventMulticaster.addContextListener(new ContextStartEventListener());
        eventMulticaster.addContextListener(new ContextRunningEventListener());
        eventMulticaster.addContextListener(new ContextDestroyEventListener());

        ((SimpleApplicationEventMulticaster) eventMulticaster).setAsync(true);

        // 发射容器启动事件ContextStartEvent
        eventMulticaster.multicastEvent(new ContextStartEvent());
        // 发射容器正在运行事件ContextRunningEvent
        eventMulticaster.multicastEvent(new ContextRunningEvent());
        // 发射容器正在运行事件ContextDestroyEvent
        eventMulticaster.multicastEvent(new ContextDestroyEvent());
        // 这里没明白在没有用CountDownLatch的情况下为何主线程退出，非后台线程的子线程也会退出？？？为了测试，所有先用CountDownLatch锁住main线程先
        // 经过测试，原来是因为用了junit的方法，test方法线程退出后，test方法线程产生的非后台线程也随之退出，而下面的main方法启动的非后台线程则不会
        // TODO 这是为什么呢？？？难道是A子线程（非main线程）启动的B子线程会随着A子线程退出而退出？还没验证
        CountDownLatch countDownLatch = new CountDownLatch(1);
        countDownLatch.await();

    }

    public static void main(String[] args) throws InterruptedException {
        // 新建SimpleApplicationEventMulticaster对象，并添加容器生命周期监听器
        ApplicationEventMulticaster eventMulticaster = new SimpleApplicationEventMulticaster();
        eventMulticaster.addContextListener(new ContextStartEventListener());
        eventMulticaster.addContextListener(new ContextRunningEventListener());
        eventMulticaster.addContextListener(new ContextDestroyEventListener());

        ((SimpleApplicationEventMulticaster) eventMulticaster).setAsync(true);

        // 发射容器启动事件ContextStartEvent
        eventMulticaster.multicastEvent(new ContextStartEvent());
        // 发射容器正在运行事件ContextRunningEvent
        eventMulticaster.multicastEvent(new ContextRunningEvent());
        // 发射容器正在运行事件ContextDestroyEvent
        eventMulticaster.multicastEvent(new ContextDestroyEvent());

    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

通过运行测试方法testContextLifecycleEventInSync()，运行结果如下截图：

![img](https://user-gold-cdn.xitu.io/2020/2/11/1703314099333a6b?w=621&h=128&f=png&s=12253)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

# 3，结语

好了，自定义事件驱动编程的简单demo就已经实现了。

这只是spring事件机制源码分析的前置文章，真正的源码分析请见下一篇博文：

**[Spring事件相关类关系源码解析--Spring的事件机制源码分析（二](https://blog.csdn.net/biaolianlao0449/article/details/104246732)**）

**原创不易，帮忙点个赞呗。**

\----------------------------------------------------------------------

微信公众号：**源码笔记**
探讨更多源码知识,关注“源码笔记”微信公众号,每周持续推出SpringBoot,Spring,Mybatis,Dubbo,RocketMQ,Jdk 和Netty等源码系列文章。

![img](https://user-gold-cdn.xitu.io/2020/2/15/17046e9b1cf0506e?w=258&h=258&f=jpeg&s=26882)