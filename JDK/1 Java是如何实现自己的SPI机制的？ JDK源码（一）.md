**注：该源码分析对应JDK版本为1.8**

# 1 引言

这是【源码笔记】的JDK源码解读的第一篇文章，本篇我们来探究Java的SPI机制的相关源码。

# 2 什么是SPI机制

那么，什么是SPI机制呢？

SPI是Service Provider Interface 的简称，即**服务提供者接口**的意思。根据字面意思我们可能还有点困惑，SPI说白了就是一种扩展机制，我们在相应配置文件中定义好某个接口的实现类，然后再根据这个接口去这个配置文件中加载这个实例类并实例化，其实SPI就是这么一个东西。说到SPI机制，我们最常见的就是Java的SPI机制，此外，还有Dubbo和SpringBoot自定义的SPI机制。

有了SPI机制，那么就为一些框架的灵活扩展提供了可能，而不必将框架的一些实现类写死在代码里面。

那么，某些框架是如何利用SPI机制来做到灵活扩展的呢？下面举几个栗子来阐述下：

1. **JDBC驱动加载案例**：利用Java的SPI机制，我们可以根据不同的数据库厂商来引入不同的JDBC驱动包；
2. **SpringBoot的SPI机制**：我们可以在`spring.factories`中加上我们自定义的自动配置类，事件监听器或初始化器等；
3. **Dubbo的SPI机制**：Dubbo更是把SPI机制应用的**淋漓尽致**，Dubbo基本上自身的每个功能点都提供了扩展点，比如提供了集群扩展，路由扩展和负载均衡扩展等差不多接近30个扩展点。如果Dubbo的某个内置实现不符合我们的需求，那么我们只要利用其SPI机制将我们的实现替换掉Dubbo的实现即可。

上面的三个栗子先让我们直观感受下某些框架利用SPI机制是如何做到灵活扩展的。


# 3 如何使用Java的SPI？

我们先来看看如何使用Java自带的SPI。
先定义一个`Developer`接口

```java
// Developer.java

package com.ymbj.spi;

public interface Developer {
    void sayHi();
}
```

再定义两个`Developer`接口的两个实现类：

```java
// JavaDeveloper.java

package com.ymbj.spi;

public class JavaDeveloper implements Developer {

    @Override
    public void sayHi() {
        System.out.println("Hi, I am a Java Developer.");
    }
}
```

```java
// PythonDeveloper.java

package com.ymbj.spi;

public class PythonDeveloper implements Developer {

    @Override
    public void sayHi() {
        System.out.println("Hi, I am a Python Developer.");
    }
}
```

然后再在项目`resources`目录下新建一个`META-INF/services`文件夹，然后再新建一个以`Developer`接口的全限定名命名的文件，文件内容为：

```java
// com.ymbj.spi.Developer文件

com.ymbj.spi.JavaDeveloper
com.ymbj.spi.PythonDeveloper
```

最后我们再新建一个测试类`JdkSPITest`：

```java
// JdkSPITest.java

public class JdkSPITest {

    @Test
    public void testSayHi() throws Exception {
        ServiceLoader<Developer> serviceLoader = ServiceLoader.load(Developer.class);
        serviceLoader.forEach(Developer::sayHi);
    }
}
```

运行上面那个测试类，运行成功结果如下截图所示：

![](https://user-gold-cdn.xitu.io/2020/3/28/1712015422d3ad09?w=438&h=81&f=png&s=5007)

由上面简单的Demo我们知道了如何使用Java的SPI机制来实现扩展点加载，下面推荐一篇文章[JAVA拾遗--关于SPI机制](https://mp.weixin.qq.com/s/Y-PFZwzSORsznJYRfiM3DA),通过这篇文章,相信大家对Java的SPI会有一个比较深刻的理解，特别是JDBC加载驱动这方面。

# 4 Java的SPI机制的源码解读

通过前面扩展`Developer`接口的简单Demo，我们看到Java的SPI机制实现跟`ServiceLoader`这个类有关，那么我们先来看下`ServiceLoader`的类结构代码：

```java
// ServiceLoader实现了【Iterable】接口
public final class ServiceLoader<S>
    implements Iterable<S>{
    private static final String PREFIX = "META-INF/services/";
    // The class or interface representing the service being loaded
    private final Class<S> service;
    // The class loader used to locate, load, and instantiate providers
    private final ClassLoader loader;
    // The access control context taken when the ServiceLoader is created
    private final AccessControlContext acc;
    // Cached providers, in instantiation order
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();
    // The current lazy-lookup iterator
    private LazyIterator lookupIterator;
    // 构造方法
    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        reload();
    }
	
    // ...暂时省略相关代码
    
    // ServiceLoader的内部类LazyIterator,实现了【Iterator】接口
    // Private inner class implementing fully-lazy provider lookup
    private class LazyIterator
        implements Iterator<S>{
        Class<S> service;
        ClassLoader loader;
        Enumeration<URL> configs = null;
        Iterator<String> pending = null;
        String nextName = null;

        private LazyIterator(Class<S> service, ClassLoader loader) {
            this.service = service;
            this.loader = loader;
        }
        // 覆写Iterator接口的hasNext方法
        public boolean hasNext() {
            // ...暂时省略相关代码
        }
        // 覆写Iterator接口的next方法
        public S next() {
            // ...暂时省略相关代码
        }
        // 覆写Iterator接口的remove方法
        public void remove() {
            // ...暂时省略相关代码
        }

    }

    // 覆写Iterable接口的iterator方法，返回一个迭代器
    public Iterator<S> iterator() {
        // ...暂时省略相关代码
    }

    // ...暂时省略相关代码

}
```

可以看到，`ServiceLoader`实现了`Iterable`接口，覆写其`iterator`方法能产生一个迭代器；同时`ServiceLoader`有一个内部类`LazyIterator`，而`LazyIterator`又实现了`Iterator`接口，说明`LazyIterator`是一个迭代器。

## 4.1 ServiceLoader.load方法，为加载服务提供者实现类做前期准备

那么我们现在开始探究Java的SPI机制的源码，
先来看`JdkSPITest`的第一句代码`ServiceLoader<Developer> serviceLoader = ServiceLoader.load(Developer.class);`中的`ServiceLoader.load(Developer.class);`的源码：

```java
// ServiceLoader.java

public static <S> ServiceLoader<S> load(Class<S> service) {
    //获取当前线程上下文类加载器 
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    // 将service接口类和线程上下文类加载器作为参数传入，继续调用load方法
    return ServiceLoader.load(service, cl);
}
```

我们再来看下`ServiceLoader.load(service, cl);`方法：

```java
// ServiceLoader.java

public static <S> ServiceLoader<S> load(Class<S> service,
                                        ClassLoader loader)
{
    // 将service接口类和线程上下文类加载器作为构造参数，新建了一个ServiceLoader对象
    return new ServiceLoader<>(service, loader);
}
```

继续看`new ServiceLoader<>(service, loader);`是如何构建的？

```java
// ServiceLoader.java

private ServiceLoader(Class<S> svc, ClassLoader cl) {
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
    reload();
}
```

可以看到在构建`ServiceLoader`对象时除了给其成员属性赋值外，还调用了`reload`方法：

```java
// ServiceLoader.java

public void reload() {
    providers.clear();
    lookupIterator = new LazyIterator(service, loader);
}
```

可以看到在`reload`方法中又新建了一个`LazyIterator`对象，然后赋值给`lookupIterator`。

```java
// ServiceLoader$LazyIterator.java

private LazyIterator(Class<S> service, ClassLoader loader) {
    this.service = service;
    this.loader = loader;
}
```

可以看到在构建`LazyIterator`对象时，也只是给其成员变量`service`和`loader`属性赋值呀，我们一路源码跟下来，也没有看到去`META-INF/services`文件夹加载`Developer`接口的实现类！这就奇怪了，我们都被`ServiceLoader`的`load`方法名骗了。

还记得分析前面的代码时新建了一个`LazyIterator`对象吗？`Lazy`顾名思义是**懒**的意思，`Iterator`就是迭代的意思。我们此时猜测那么`LazyIterator`对象的作用应该就是在迭代的时候再去加载`Developer`接口的实现类了。

## 4.2 ServiceLoader.iterator方法，实现服务提供者实现类的懒加载

我们现在再来看`JdkSPITest`的第二句代码`serviceLoader.forEach(Developer::sayHi);`，执行这句代码后最终会调用`serviceLoader`的`iterator`方法：

```java
// serviceLoader.java

public Iterator<S> iterator() {
    return new Iterator<S>() {

        Iterator<Map.Entry<String,S>> knownProviders
            = providers.entrySet().iterator();

        public boolean hasNext() {
            if (knownProviders.hasNext())
                return true;
            // 调用lookupIterator即LazyIterator的hasNext方法
            // 可以看到是委托给LazyIterator的hasNext方法来实现
            return lookupIterator.hasNext();
        }

        public S next() {
            if (knownProviders.hasNext())
                return knownProviders.next().getValue();
            // 调用lookupIterator即LazyIterator的next方法
            // 可以看到是委托给LazyIterator的next方法来实现
            return lookupIterator.next();
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    };
}
```

可以看到调用`serviceLoader`的`iterator`方法会返回一个匿名的迭代器对象，而这个匿名迭代器对象其实相当于一个门面类，其覆写的`hasNext`和`next`方法又分别委托`LazyIterator`的`hasNext`和`next`方法来实现了。

我们继续调试，发现接下来会进入`LazyIterator`的`hasNext`方法：

```java
// serviceLoader$LazyIterator.java

public boolean hasNext() {
    if (acc == null) {
        // 调用hasNextService方法
        return hasNextService();
    } else {
        PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
            public Boolean run() { return hasNextService(); }
        };
        return AccessController.doPrivileged(action, acc);
    }
}
```

继续跟进`hasNextService`方法：

```java
// serviceLoader$LazyIterator.java

private boolean hasNextService() {
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            // PREFIX = "META-INF/services/"
            // service.getName()即接口的全限定名
            // 还记得前面的代码构建LazyIterator对象时已经给其成员属性service赋值吗
            String fullName = PREFIX + service.getName();
            // 加载META-INF/services/目录下的接口文件中的服务提供者类
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                // 还记得前面的代码构建LazyIterator对象时已经给其成员属性loader赋值吗
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        // 返回META-INF/services/目录下的接口文件中的服务提供者类并赋值给pending属性
        pending = parse(service, configs.nextElement());
    }
    // 然后取出一个全限定名赋值给LazyIterator的成员变量nextName
    nextName = pending.next();
    return true;
}
```

可以看到在执行`LazyIterator`的`hasNextService`方法时最终将去`META-INF/services/`目录下加载接口文件的内容即加载服务提供者实现类的全限定名，然后取出一个服务提供者实现类的全限定名赋值给`LazyIterator`的成员变量`nextName`。到了这里，我们就明白了`LazyIterator`的作用真的是懒加载，在用到的时候才会去加载。

> **思考**：为何这里要用懒加载呢？懒加载的思想是怎样的呢？懒加载有啥好处呢？你还能举出其他懒加载的案例吗？

同样，执行完`LazyIterator`的`hasNext`方法后，会继续执行`LazyIterator`的`next`方法：

```java
// serviceLoader$LazyIterator.java

public S next() {
    if (acc == null) {
        // 调用nextService方法
        return nextService();
    } else {
        PrivilegedAction<S> action = new PrivilegedAction<S>() {
            public S run() { return nextService(); }
        };
        return AccessController.doPrivileged(action, acc);
    }
}
```

我们继续跟进`nextService`方法：

```java
// serviceLoader$LazyIterator.java

private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    // 还记得在hasNextService方法中为nextName赋值过服务提供者实现类的全限定名吗
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        // 【1】去classpath中根据传入的类加载器和服务提供者实现类的全限定名去加载服务提供者实现类
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn  + " not a subtype");
    }
    try {
        // 【2】实例化刚才加载的服务提供者实现类，并进行转换
        S p = service.cast(c.newInstance());
        // 【3】最终将实例化后的服务提供者实现类放进providers集合
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error();          // This cannot happen
}
```

可以看到`LazyIterator`的`nextService`方法最终将实例化之前加载的服务提供者实现类，并放进`providers`集合中，随后再调用服务提供者实现类的方法（比如这里指`JavaDeveloper`的`sayHi`方法）。注意，这里是加载一个服务提供者实现类后，若`main`函数中有调用该服务提供者实现类的方法的话，紧接着会调用其方法；然后继续实例化下一个服务提供者类。

> **设计模式**：可以看到，Java的SPI机制实现代码中应用了迭代器模式，迭代器模式屏蔽了各种存储对象的内部结构差异，提供一个统一的视图来遍历各个存储对象（存储对象可以为集合，数组等）。`java.util.Iterator`也是迭代器模式的实现：同时Java的各个集合类一般实现了`Iterable`接口，实现了其`iterator`方法从而获得`Iterator`接口的实现类对象（一般为集合内部类），然后再利用`Iterator`对象的实现类的`hasNext`和`next`方法来遍历集合元素。


# 5 JDBC驱动加载源码解读
前面分析了Java的SPI机制的源码实现，现在我们再来看下Java的SPI机制的实际案例的应用。

我们都知道，JDBC驱动加载是Java的SPI机制的典型应用案例。JDBC主要提供了一套接口规范，而这套规范的api在java的核心库（`rt.jar`）中实现，而不同的数据库厂商只要编写符合这套JDBC接口规范的驱动代码，那么就可以用Java语言来连接数据库了。

java的核心库（`rt.jar`）中跟JDBC驱动加载的最核心的接口和类分别是`java.sql.Driver`接口和`java.sql.DriverManager`类，其中`java.sql.Driver`是各个数据库厂商的驱动类要实现的接口，而`DriverManager`是用来管理数据库的驱动类的，值得注意的是`DriverManager`这个类有一个`registeredDrivers`集合属性，用来存储数据库的驱动类。
```java
// DriverManager.java

// List of registered JDBC drivers
private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();
```

这里以加载Mysql驱动为例来分析JDBC驱动加载的源码。

我们的项目引入`mysql-connector-java`依赖（这里的版本是`5.1.47`）后，那么Mysql的驱动实现类文件如下图所示：


![](https://user-gold-cdn.xitu.io/2020/3/28/17121285dd574a92?w=1425&h=382&f=png&s=103626)
可以看到Mysql的驱动包中有两个`Driver`驱动类，分别是`com.mysql.jdbc.Driver`和`com.mysql.fabric.jdbc.FabricMySQLDriver`，默认情况下一般我们只用到前者。

## 5.1 利用Java的SPI加载Mysql的驱动类

那么接下来我们就来探究下JDBC驱动加载的代码是如何实现的。

先来看一下一个简单的JDBC的测试代码：
```java
// JdbcTest.java

public class JdbcTest {
    public static void main(String[] args) {
        Connection connection = null;  
        Statement statement = null;
        ResultSet rs = null;

        try {
            // 注意：在JDBC 4.0规范中，这里可以不用再像以前那样编写显式加载数据库的代码了
            // Class.forName("com.mysql.jdbc.Driver");
            // 获取数据库连接，注意【这里将会加载mysql的驱动包】
            /***************【主线，切入点】****************/
            connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/jdbc", "root", "123456");
            // 创建Statement语句
            statement = connection.createStatement();
            // 执行查询语句
            rs = statement.executeQuery("select * from user");
            // 遍历查询结果集
            while(rs.next()){
                String name = rs.getString("name");
                System.out.println(name);
            }
        } catch(Exception e) {
            e.printStackTrace();
        } finally {
            // ...省略释放资源的代码
        }
    }
}
```
在`JdbcTest`的`main`函数调用`DriverManager`的`getConnection`方法时，此时必然会先执行`DriverManager`类的静态代码块的代码，然后再执行`getConnection`方法，那么先来看下`DriverManager`的静态代码块：
```java
// DriverManager.java

static {
    // 加载驱动实现类
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}
```
继续跟进`loadInitialDrivers`的代码：
```java
// DriverManager.java

private static void loadInitialDrivers() {
    String drivers;
    try {
        drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
            public String run() {
                return System.getProperty("jdbc.drivers");
            }
        });
    } catch (Exception ex) {
        drivers = null;
    }
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            // 来到这里，是不是感觉似曾相识，对，没错，我们在前面的JdkSPITest代码中执行过下面的两句代码
            // 这句代码前面已经分析过，这里不会真正加载服务提供者实现类
            // 而是实例化一个ServiceLoader对象且实例化一个LazyIterator对象用于懒加载
            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
            // 调用ServiceLoader的iterator方法，在迭代的同时，也会去加载并实例化META-INF/services/java.sql.Driver文件
            // 的com.mysql.jdbc.Driver和com.mysql.fabric.jdbc.FabricMySQLDriver两个驱动类
            /****************【主线，重点关注】**********************/
            Iterator<Driver> driversIterator = loadedDrivers.iterator();
            try{
                while(driversIterator.hasNext()) {
                    driversIterator.next();
                }
            } catch(Throwable t) {
            // Do nothing
            }
            return null;
        }
    });

    println("DriverManager.initialize: jdbc.drivers = " + drivers);

    if (drivers == null || drivers.equals("")) {
        return;
    }
    String[] driversList = drivers.split(":");
    println("number of Drivers:" + driversList.length);
    for (String aDriver : driversList) {
        try {
            println("DriverManager.Initialize: loading " + aDriver);
            Class.forName(aDriver, true,
                    ClassLoader.getSystemClassLoader());
        } catch (Exception ex) {
            println("DriverManager.Initialize: load failed: " + ex);
        }
    }
}
```
在上面的代码中，我们可以看到Mysql的驱动类加载主要是利用Java的SPI机制实现的，即利用`ServiceLoader`来实现加载并实例化Mysql的驱动类。

## 5.2 注册Mysql的驱动类

那么，上面的代码只是Mysql驱动类的加载和实例化，**那么，驱动类又是如何被注册进`DriverManager`的`registeredDrivers`集合的呢？**

这时，我们注意到`com.mysql.jdbc.Driver`类里面也有个静态代码块，即实例化该类时肯定会触发该静态代码块代码的执行，那么我们直接看下这个静态代码块做了什么事情：
```java
// com.mysql.jdbc.Driver.java

// Register ourselves with the DriverManager
static {
    try {
        // 将自己注册进DriverManager类的registeredDrivers集合
        java.sql.DriverManager.registerDriver(new Driver());
    } catch (SQLException E) {
        throw new RuntimeException("Can't register driver!");
    }
}
```
可以看到，原来就是Mysql驱动类`com.mysql.jdbc.Driver`在实例化的时候，利用执行其静态代码块的时机时将自己注册进`DriverManager`的`registeredDrivers`集合中。

好，继续跟进`DriverManager`的`registerDriver`方法：
```java
// DriverManager.java

public static synchronized void registerDriver(java.sql.Driver driver)
    throws SQLException {
    // 继续调用registerDriver方法
    registerDriver(driver, null);
}

public static synchronized void registerDriver(java.sql.Driver driver,
        DriverAction da)
    throws SQLException {

    /* Register the driver if it has not already been added to our list */
    if(driver != null) {
        // 将driver驱动类实例注册进registeredDrivers集合
        registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
    } else {
        // This is for compatibility with the original DriverManager
        throw new NullPointerException();
    }
    println("registerDriver: " + driver);
}
```
分析到了这里，我们就明白了Java的SPI机制是如何加载Mysql的驱动类的并如何将Mysql的驱动类注册进`DriverManager`的`registeredDrivers`集合中的。

## 5.3 使用之前注册的Mysql驱动类连接数据库

**既然Mysql的驱动类已经被注册进来了，那么何时会被用到呢？**

我们要连接Mysql数据库，自然需要用到Mysql的驱动类，对吧。此时我们回到JDBC的测试代码`JdbcTest`类的`connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/jdbc", "root", "123456");`这句代码中，看一下`getConnection`的源码：

```java
// DriverManager.java

@CallerSensitive
public static Connection getConnection(String url,
    String user, String password) throws SQLException {
    java.util.Properties info = new java.util.Properties();

    if (user != null) {
        info.put("user", user);
    }
    if (password != null) {
        info.put("password", password);
    }
    // 继续调用getConnection方法来连接数据库
    return (getConnection(url, info, Reflection.getCallerClass()));
}
```
继续跟进`getConnection`方法：
```java
// DriverManager.java

private static Connection getConnection(
        String url, java.util.Properties info, Class<?> caller) throws SQLException {
        
        ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
        synchronized(DriverManager.class) {
            // synchronize loading of the correct classloader.
            if (callerCL == null) {
                callerCL = Thread.currentThread().getContextClassLoader();
            }
        }
        if(url == null) {
            throw new SQLException("The url cannot be null", "08001");
        }
        println("DriverManager.getConnection(\"" + url + "\")");
        // Walk through the loaded registeredDrivers attempting to make a connection.
        // Remember the first exception that gets raised so we can reraise it.
        SQLException reason = null;
        // 遍历registeredDrivers集合，注意之前加载的Mysql驱动类实例被注册进这个集合
        for(DriverInfo aDriver : registeredDrivers) {
            // If the caller does not have permission to load the driver then
            // skip it.
            // 判断有无权限
            if(isDriverAllowed(aDriver.driver, callerCL)) {
                try {
                    println("    trying " + aDriver.driver.getClass().getName());
                    // 利用Mysql驱动类来连接数据库
                    /*************【主线，重点关注】*****************/
                    Connection con = aDriver.driver.connect(url, info);
                    // 只要连接上，那么加载的其余驱动类比如FabricMySQLDriver将会忽略，因为下面直接返回了
                    if (con != null) {
                        // Success!
                        println("getConnection returning " + aDriver.driver.getClass().getName());
                        return (con);
                    }
                } catch (SQLException ex) {
                    if (reason == null) {
                        reason = ex;
                    }
                }

            } else {
                println("    skipping: " + aDriver.getClass().getName());
            }

        }

        // if we got here nobody could connect.
        if (reason != null)    {
            println("getConnection failed: " + reason);
            throw reason;
        }

        println("getConnection: no suitable driver found for "+ url);
        throw new SQLException("No suitable driver found for "+ url, "08001");
    }
```
可以看到，`DriverManager`的`getConnection`方法会从`registeredDrivers`集合中拿出刚才加载的Mysql驱动类来连接数据库。

好了，到了这里，JDBC驱动加载的源码就基本分析完了。
# 6 线程上下文类加载器
前面基本分析完了JDBC驱动加载的源码，但是还有一个很重要的知识点还没讲解，那就是破坏类加载机制的双亲委派模型的**线程上下文类加载器**。

我们都知道，JDBC规范的相关类（比如前面的`java.sql.Driver`和`java.sql.DriverManager`）都是在Jdk的`rt.jar`包下，意味着这些类将由启动类加载器(BootstrapClassLoader)加载；而Mysql的驱动类由外部数据库厂商实现，当驱动类被引进项目时也是位于项目的`classpath`中,此时启动类加载器肯定是不可能加载这些驱动类的呀，此时该怎么办？

由于类加载机制的双亲委派模型在这方面的缺陷，因此只能打破双亲委派模型了。因为项目`classpath`中的类是由应用程序类加载器(AppClassLoader)来加载，所以我们可否"逆向"让启动类加载器委托应用程序类加载器去加载这些外部数据库厂商的驱动类呢？如果可以，我们怎样才能做到让启动类加载器委托应用程序类加载器去加载
`classpath`中的类呢？

答案肯定是可以的，我们可以将应用程序类加载器设置进线程里面，即线程里面新定义一个类加载器的属性`contextClassLoader`，然后在某个时机将应用程序类加载器设置进线程的`contextClassLoader`这个属性里面，如果没有设置的话，那么默认就是应用程序类加载器。然后启动类加载器去加载`java.sql.Driver`和`java.sql.DriverManager`等类时，同时也会从当前线程中取出`contextClassLoader`即应用程序类加载器去`classpath`中加载外部厂商提供的JDBC驱动类。因此，通过破坏类加载机制的双亲委派模型，利用**线程上下文类加载器**完美的解决了该问题。

此时我们再回过头来看下**在加载Mysql驱动时是什么时候获取的线程上下文类加载器呢？**

答案就是在`DriverManager`的`loadInitialDrivers`方法调用了`ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);`这句代码，而取出线程上下文类加载器就是在`ServiceLoader`的`load`方法中取出：
```java

public static <S> ServiceLoader<S> load(Class<S> service) {
    // 取出线程上下文类加载器取出的是contextClassLoader，而contextClassLoader装的应用程序类加载器
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    // 把刚才取出的线程上下文类加载器作为参数传入，用于后去加载classpath中的外部厂商提供的驱动类
    return ServiceLoader.load(service, cl);
}
```
因此，到了这里，我们就明白了线程上下文类加载器在加载JDBC驱动包中充当的作用了。此外，我们应该知道，Java的绝大部分涉及SPI的加载都是利用线程上下文类加载器来完成的，比如JNDI,JCE,JBI等。

> **扩展**：打破类加载机制的双亲委派模型的还有代码的热部署等，另外，Tomcat的类加载机制也值得一读。

注：若有些小伙伴对类加载机制的双亲委派模型不清楚的话，推荐[完全理解双亲委派模型与自定义 ClassLoader](https://mp.weixin.qq.com/s/Hy4OSYI8_s0tlEmMfZr-UQ)这篇文了解下。
# 7 扩展：Dubbo的SPI机制
前面也讲到Dubbo框架身上处处是SPI机制的应用，可以说处处都是扩展点，真的是把SPI机制应用的淋漓尽致。但是Dubbo没有采用默认的Java的SPI机制，而是自己实现了一套SPI机制。

那么，**Dubbo为什么没有采用Java的SPI机制呢？**

原因主要有两个：

1. Java的SPI机制会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源;
2. Java的SPI机制没有Ioc和AOP的支持，因此Dubbo用了自己的SPI机制：增加了对扩展点IoC和AOP的支持，一个扩展点可以直接setter注入其它扩展点。

由于以上原因，Dubbo自定义了一套SPI机制，用于加载自己的扩展点。关于Dubbo的SPI机制这里不再详述，感兴趣的小伙伴们可以去Dubbo官网看看是如何扩展Dubbo的SPI的？还有其官网也有Duboo的SPI的源码分析文章。
# 8 小结

好了，Java的SPI机制就解读到这里了，先将前面的知识点再总结下：
1. Java的SPI机制的使用；
2. Java的SPI机制的原理；
3. JDBC驱动的加载原理；
4. 线程上下文类加载器在JDBC驱动加载中的作用；
5. 简述了Duboo的SPI机制。

**原创不易，帮忙点个赞呗！**

由于笔者水平有限，若文中有错误还请指出，谢谢。

**参考**：

1，http://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html

2，《深入理解Java虚拟机》
