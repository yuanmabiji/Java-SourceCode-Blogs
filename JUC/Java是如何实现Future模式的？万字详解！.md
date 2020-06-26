JDK1.8源码分析项目（中文注释）Github地址：

https://github.com/yuanmabiji/jdk1.8-sourcecode-blogs
# 1 Future是什么？
先举个例子，我们平时网购买东西，下单后会生成一个订单号，然后商家会根据这个订单号发货，发货后又有一个快递单号，然后快递公司就会根据这个快递单号将网购东西快递给我们。在这一过程中，这一系列的单号都是我们收货的重要凭证。

因此，JDK的Future就类似于我们网购买东西的单号，当我们执行某一耗时的任务时，我们可以另起一个线程异步去执行这个耗时的任务，同时我们可以干点其他事情。当事情干完后我们再根据future这个"单号"去提取耗时任务的执行结果即可。因此Future也是多线程中的一种应用模式。

> **扩展**: 说起多线程，那么Future又与Thread有什么区别呢？最重要的区别就是Thread是没有返回结果的，而Future模式是有返回结果的。

# 2 如何使用Future
前面搞明白了什么是Future，下面我们再来举个简单的例子看看如何使用Future。

假如现在我们要打火锅，首先我们要准备两样东西：把水烧开和准备食材。因为烧开水是一个比较漫长的过程（相当于耗时的业务逻辑），因此我们可以一边烧开水（相当于另起一个线程），一边准备火锅食材（主线程），等两者都准备好了我们就可以开始打火锅了。

```java
// DaHuoGuo.java

public class DaHuoGuo {
	public static void main(String[] args) throws Exception {
		FutureTask<String> futureTask = new FutureTask<>(new Callable<String>() {
			@Override
			public String call() throws Exception {
				System.out.println(Thread.currentThread().getName() + ":" + "开始烧开水...");
				// 模拟烧开水耗时
				Thread.sleep(2000);
				System.out.println(Thread.currentThread().getName() + ":"  + "开水已经烧好了...");
				return "开水";
			}
		});

		Thread thread = new Thread(futureTask);
		thread.start();

		// do other thing
		System.out.println(Thread.currentThread().getName() + ":"  + " 此时开启了一个线程执行future的逻辑（烧开水），此时我们可以干点别的事情（比如准备火锅食材）...");
		// 模拟准备火锅食材耗时
		Thread.sleep(3000);
		System.out.println(Thread.currentThread().getName() + ":"  + "火锅食材准备好了");
		String shicai = "火锅食材";

		// 开水已经稍好，我们取得烧好的开水
		String boilWater = futureTask.get();

		System.out.println(Thread.currentThread().getName() + ":"  + boilWater + "和" + shicai + "已经准备好，我们可以开始打火锅啦");
	}
}
```
执行结果如下截图，符合我们的预期：
![](https://user-gold-cdn.xitu.io/2020/6/21/172d728d9ad5b1f1?w=1008&h=150&f=png&s=25755)

从以上代码中可以看到，我们使用Future主要有以下步骤：
1. 新建一个`Callable`匿名函数实现类对象，我们的业务逻辑在`Callable`的`call`方法中实现，其中Callable的泛型是返回结果类型；
2. 然后把`Callable`匿名函数对象作为`FutureTask`的构造参数传入，构建一个`futureTask`对象；
3. 然后再把`futureTask`对象作为`Thread`构造参数传入并开启这个线程执行去执行业务逻辑；
4. 最后我们调用`futureTask`对象的`get`方法得到业务逻辑执行结果。

可以看到跟Future使用有关的JDK类主要有`FutureTask`和`Callable`两个，下面主要对`FutureTask`进行源码分析。
> **扩展**： 还有一种使用`Future`的方式是将`Callable`实现类提交给线程池执行的方式，这里不再介绍，自行百度即可。

# 3 FutureTask类结构分析
我们先来看下`FutureTask`的类结构：

![](https://user-gold-cdn.xitu.io/2020/6/21/172d73948efa22e2?w=299&h=299&f=png&s=8011)
可以看到`FutureTask`实现了`RunnableFuture`接口，而`RunnableFuture`接口又继承了`Future`和`Runnable`接口。因为`FutureTask`间接实现了`Runnable`接口，因此可以作为任务被线程`Thread`执行；此外，**最重要的一点**就是`FutureTask`还间接实现了`Future`接口，因此还可以获得任务执行的结果。下面我们就来简单看看这几个接口的相关`api`。
```java
// Runnable.java

@FunctionalInterface
public interface Runnable {
    // 执行线程任务
    public abstract void run();
}
```
`Runnable`没啥好说的，相信大家都已经很熟悉了。
```java
// Future.java

public interface Future<V> {
    /**
     * 尝试取消线程任务的执行，分为以下几种情况：
     * 1）如果线程任务已经完成或已经被取消或其他原因不能被取消，此时会失败并返回false；
     * 2）如果任务还未开始执行，此时执行cancel方法，那么任务将被取消执行，此时返回true；TODO 此时对应任务状态state的哪种状态？？？不懂！！
     * 3）如果任务已经开始执行，那么mayInterruptIfRunning这个参数将决定是否取消任务的执行。
     *    这里值得注意的是，cancel(true)实质并不能真正取消线程任务的执行，而是发出一个线程
     *    中断的信号，一般需要结合Thread.currentThread().isInterrupted()来使用。
     */
    boolean cancel(boolean mayInterruptIfRunning);
    /**
     * 判断任务是否被取消，在执行任务完成前被取消，此时会返回true
     */
    boolean isCancelled();
    /**
     * 这个方法不管任务正常停止，异常还是任务被取消，总是返回true。
     */
    boolean isDone();
    /**
     * 获取任务执行结果，注意是阻塞等待获取任务执行结果。
     */
    V get() throws InterruptedException, ExecutionException;
    /**
     * 获取任务执行结果，注意是阻塞等待获取任务执行结果。
     * 只不过在规定的时间内未获取到结果，此时会抛出超时异常
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
`Future`接口象征着异步执行任务的结果即执行一个耗时任务完全可以另起一个线程执行，然后此时我们可以去做其他事情，做完其他事情我们再调用`Future.get()`方法获取结果即可，此时若异步任务还没结束，此时会一直阻塞等待，直到异步任务执行完获取到结果。

```java
// RunnableFuture.java

public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```
`RunnableFuture`是`Future`和`Runnable`接口的组合，即这个接口表示又可以被线程异步执行，因为实现了`Runnable`接口，又可以获得线程异步任务的执行结果，因为实现了`Future`接口。因此解决了`Runnable`异步任务没有返回结果的缺陷。

接下来我们来看下`FutureTask`，`FutureTask`实现了`RunnableFuture`接口，因此是`Future`和`Runnable`接口的具体实现类，是一个可被取消的异步线程任务，提供了`Future`的基本实现，即异步任务执行后我们能够获取到异步任务的执行结果，是我们接下来分析的重中之重。`FutureTask`可以包装一个`Callable`和`Runnable`对象，此外,`FutureTask`除了可以被线程执行外，还可以被提交给线程池执行。

我们先看下`FutureTask`类的`api`，其中重点方法已经红框框出。

![](https://user-gold-cdn.xitu.io/2020/6/25/172e98c0611e7c60?w=389&h=429&f=png&s=27506)
上图中`FutureTask`的`run`方法是被线程异步执行的方法，`get`方法即是取得异步任务执行结果的方法，还有`cancel`方法是取消任务执行的方法。接下来我们主要对这三个方法进行重点分析。
> **思考**：
1. `FutureTask`覆写的`run`方法的返回类型依然是`void`，表示没有返回值，那么`FutureTask`的`get`方法又是如何获得返回值的呢？
2. `FutureTask`的`cancel`方法能真正取消线程异步任务的执行么？什么情况下能取消？

因为`FutureTask`异步任务执行结果还跟`Callable`接口有关，因此我们再来看下`Callable`接口：
```java
// Callable.java

@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     */
    V call() throws Exception;
}
```
我们都知道，`Callable<V>`接口和`Runnable`接口都可以被提交给线程池执行，唯一不同的就是`Callable<V>`接口是有返回结果的，其中的泛型`V`就是返回结果，而`Runnable`接口是没有返回结果的。
> **思考**： 一般情况下，`Runnable`接口实现类才能被提交给线程池执行，为何`Callable`接口实现类也可以被提交给线程池执行？想想线程池的`submit`方法内部有对`Callable`做适配么？

# 4 FutureTask源码分析

## 4.1 FutureTask成员变量
我们首先来看下`FutureTask`的成员变量有哪些,理解这些成员变量对后面的源码分析非常重要。
```java
// FutureTask.java

/** 封装的Callable对象，其call方法用来执行异步任务 */
private Callable<V> callable;
/** 在FutureTask里面定义一个成员变量outcome，用来装异步任务的执行结果 */
private Object outcome; // non-volatile, protected by state reads/writes
/** 用来执行callable任务的线程 */
private volatile Thread runner;
/** 线程等待节点，reiber stack的一种实现 */
private volatile WaitNode waiters;
/** 任务执行状态 */
private volatile int state;

// Unsafe mechanics
private static final sun.misc.Unsafe UNSAFE;
// 对应成员变量state的偏移地址
private static final long stateOffset;
// 对应成员变量runner的偏移地址
private static final long runnerOffset;
// 对应成员变量waiters的偏移地址
private static final long waitersOffset;
```
这里我们要重点关注下`FutureTask`的`Callable`成员变量，因为`FutureTask`的异步任务最终是委托给`Callable`去实现的。

> **思考**：
1. `FutureTask`的成员变量`runner`,`waiters`和`state`都被`volatile`修饰，我们可以思考下为什么这三个成员变量需要被`volatile`修饰，而其他成员变量又不用呢？`volatile`关键字的作用又是什么呢？
2. 既然已经定义了成员变量`runner`,`waiters`和`state`了，此时又定义了`stateOffset`,`runnerOffset`和`waitersOffset`变量分别对应`runner`,`waiters`和`state`的偏移地址，为何要多此一举呢？

我们再来看看`stateOffset`,`runnerOffset`和`waitersOffset`变量这三个变量的初始化过程：
```java
// FutureTask.java

static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> k = FutureTask.class;
        stateOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("state"));
        runnerOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("runner"));
        waitersOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("waiters"));
    } catch (Exception e) {
        throw new Error(e);
    }
    }
```
## 4.2 FutureTask的状态变化
前面讲了`FutureTask`的成员变量，有一个表示**状态**的成员变量`state`我们要重点关注下，`state`变量表示任务执行的状态。
```java
// FutureTask.java

/** 任务执行状态 */
private volatile int state;
/** 任务新建状态 */
private static final int NEW          = 0;
/** 任务正在完成状态，是一个瞬间过渡状态 */
private static final int COMPLETING   = 1;
/** 任务正常结束状态 */
private static final int NORMAL       = 2;
/** 任务执行异常状态 */
private static final int EXCEPTIONAL  = 3;
/** 任务被取消状态，对应cancel(false) */
private static final int CANCELLED    = 4;
/** 任务中断状态，是一个瞬间过渡状态 */
private static final int INTERRUPTING = 5;
/** 任务被中断状态，对应cancel(true) */
private static final int INTERRUPTED  = 6;
```
可以看到任务状态变量`state`有以上7种状态，0-6分别对应着每一种状态。任务状态一开始是`NEW`,然后由`FutureTask`的三个方法`set`,`setException`和`cancel`来设置状态的变化，其中状态变化有以下四种情况：

1. `NEW -> COMPLETING -> NORMAL`:这个状态变化表示异步任务的正常结束，其中`COMPLETING`是一个瞬间临时的过渡状态，由`set`方法设置状态的变化；
2. `NEW -> COMPLETING -> EXCEPTIONAL`:这个状态变化表示异步任务执行过程中抛出异常，由`setException`方法设置状态的变化；
3. `NEW -> CANCELLED`:这个状态变化表示被取消，即调用了`cancel(false)`，由`cancel`方法来设置状态变化；
4. `NEW -> INTERRUPTING -> INTERRUPTED`:这个状态变化表示被中断，即调用了`cancel(true)`，由`cancel`方法来设置状态变化。





## 4.3 FutureTask构造函数
`FutureTask`有两个构造函数，我们分别来看看：
```java
// FutureTask.java

// 第一个构造函数
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```
可以看到，这个构造函数在我们前面举的“打火锅”的例子代码中有用到，就是`Callable`成员变量赋值，在异步执行任务时再调用`Callable.call`方法执行异步任务逻辑。此外，此时给任务状态`state`赋值为`NEW`，表示任务新建状态。

我们再来看下`FutureTask`的另外一个构造函数：
```java
// FutureTask.java

// 另一个构造函数
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```
这个构造函数在执行`Executors.callable(runnable, result)`时是通过适配器`RunnableAdapter`来将`Runnable`对象`runnable`转换成`Callable`对象，然后再分别给`callable`和`state`变量赋值。

**注意**，这里我们需要记住的是`FutureTask`新建时，此时的任务状态`state`是`NEW`就好了。
## 4.4 FutureTask.run方法,用来执行异步任务
前面我们有讲到`FutureTask`间接实现了`Runnable`接口，覆写了`Runnable`接口的`run`方法，因此该覆写的`run`方法是提交给线程来执行的，同时，该`run`方法正是执行异步任务逻辑的方法，那么，执行完`run`方法又是如何保存异步任务执行的结果的呢？

我们现在着重来分析下`run`方法：
```java
// FutureTask.java

public void run() {
    // 【1】,为了防止多线程并发执行异步任务，这里需要判断线程满不满足执行异步任务的条件，有以下三种情况：
    // 1）若任务状态state为NEW且runner为null，说明还未有线程执行过异步任务，此时满足执行异步任务的条件，
    // 此时同时调用CAS方法为成员变量runner设置当前线程的值；
    // 2）若任务状态state为NEW且runner不为null，任务状态虽为NEW但runner不为null，说明有线程正在执行异步任务，
    // 此时不满足执行异步任务的条件，直接返回；
    // 1）若任务状态state不为NEW，此时不管runner是否为null，说明已经有线程执行过异步任务，此时没必要再重新
    // 执行一次异步任务，此时不满足执行异步任务的条件；
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        // 拿到之前构造函数传进来的callable实现类对象，其call方法封装了异步任务执行的逻辑
        Callable<V> c = callable;
        // 若任务还是新建状态的话，那么就调用异步任务
        if (c != null && state == NEW) {
            // 异步任务执行结果
            V result;
            // 异步任务执行成功还是始遍标志
            boolean ran;
            try {
                // 【2】，执行异步任务逻辑，并把执行结果赋值给result
                result = c.call();
                // 若异步任务执行过程中没有抛出异常，说明异步任务执行成功，此时设置ran标志为true
                ran = true;
            } catch (Throwable ex) {
                result = null;
                // 异步任务执行过程抛出异常，此时设置ran标志为false
                ran = false;
                // 【3】设置异常，里面也设置state状态的变化
                setException(ex);
            }
            // 【3】若异步任务执行成功，此时设置异步任务执行结果，同时也设置状态的变化
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        // 异步任务正在执行过程中，runner一直是非空的，防止并发调用run方法，前面有调用cas方法做判断的
        // 在异步任务执行完后，不管是正常结束还是异常结束，此时设置runner为null
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        // 线程执行异步任务后的任务状态
        int s = state;
        // 【4】如果执行了cancel(true)方法，此时满足条件，
        // 此时调用handlePossibleCancellationInterrupt方法处理中断
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```
可以看到执行异步任务的`run`方法主要分为以下四步来执行：
1. **判断线程是否满足执行异步任务的条件**：为了防止多线程并发执行异步任务，这里需要判断线程满不满足执行异步任务的条件；
2. **若满足条件，执行异步任务**：因为异步任务逻辑封装在`Callable.call`方法中，此时直接调用`Callable.call`方法执行异步任务，然后返回执行结果；
3. **根据异步任务的执行情况做不同的处理**：1） 若异步任务执行正常结束，此时调用`set(result);`来设置任务执行结果；2）若异步任务执行抛出异常，此时调用`setException(ex);`来设置异常,详细分析请见`4.4.1小节`；
4. **异步任务执行完后的善后处理工作**：不管异步任务执行成功还是失败，若其他线程有调用`FutureTask.cancel(true)`，此时需要调用`handlePossibleCancellationInterrupt`方法处理中断，详细分析请见`4.4.2小节`。

这里**值得注意**的是判断线程满不满足执行异步任务条件时,`runner`是否为`null`是调用`UNSAFE`的`CAS`方法`compareAndSwapObject`来判断和设置的，同时`compareAndSwapObject`是通过成员变量`runner`的偏移地址`runnerOffset`来给`runner`赋值的，此外，成员变量`runner`被修饰为`volatile`是在多线程的情况下， 一个线程的`volatile`修饰变量的设值能够立即刷进主存，因此值便可被其他线程可见。

### 4.4.1 FutureTask的set和setException方法
下面我们来看下当异步任务执行正常结束时，此时会调用`set(result);`方法：
```java
// FutureTask.java

protected void set(V v) {
    // 【1】调用UNSAFE的CAS方法判断任务当前状态是否为NEW，若为NEW，则设置任务状态为COMPLETING
    // 【思考】此时任务不能被多线程并发执行，什么情况下会导致任务状态不为NEW？
    // 答案是只有在调用了cancel方法的时候，此时任务状态不为NEW，此时什么都不需要做，
    // 因此需要调用CAS方法来做判断任务状态是否为NEW
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        // 【2】将任务执行结果赋值给成员变量outcome
        outcome = v;
        // 【3】将任务状态设置为NORMAL，表示任务正常结束
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        // 【4】调用任务执行完成方法，此时会唤醒阻塞的线程，调用done()方法和清空等待线程链表等
        finishCompletion();
    }
}
```
可以看到当异步任务正常执行结束后，且异步任务没有被`cancel`的情况下，此时会做以下事情：将任务执行结果保存到`FutureTask`的成员变量`outcome`中的，赋值结束后会调用`finishCompletion`方法来唤醒阻塞的线程（哪里来的阻塞线程？后面会分析），**值得注意**的是这里对应的任务状态变化是**NEW -> COMPLETING -> NORMAL**。


我们继续来看下当异步任务执行过程中抛出异常，此时会调用`setException(ex);`方法。
```java
// FutureTask.java

protected void setException(Throwable t) {
    // 【1】调用UNSAFE的CAS方法判断任务当前状态是否为NEW，若为NEW，则设置任务状态为COMPLETING
    // 【思考】此时任务不能被多线程并发执行，什么情况下会导致任务状态不为NEW？
    // 答案是只有在调用了cancel方法的时候，此时任务状态不为NEW，此时什么都不需要做，
    // 因此需要调用CAS方法来做判断任务状态是否为NEW
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        // 【2】将异常赋值给成员变量outcome
        outcome = t;
        // 【3】将任务状态设置为EXCEPTIONAL
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        // 【4】调用任务执行完成方法，此时会唤醒阻塞的线程，调用done()方法和清空等待线程链表等
        finishCompletion();
    }
}
```
可以看到`setException(Throwable t)`的代码逻辑跟前面的`set(V v)`几乎一样，不同的是任务执行过程中抛出异常，此时是将异常保存到`FutureTask`的成员变量`outcome`中，还有，**值得注意**的是这里对应的任务状态变化是**NEW -> COMPLETING -> EXCEPTIONAL**。

因为异步任务不管正常还是异常结束，此时都会调用`FutureTask`的`finishCompletion`方法来唤醒唤醒阻塞的线程，这里阻塞的线程是指我们调用`Future.get`方法时若异步任务还未执行完，此时该线程会阻塞。
```java
// FutureTask.java

private void finishCompletion() {
    // assert state > COMPLETING;
    // 取出等待线程链表头节点，判断头节点是否为null
    // 1）若线程链表头节点不为空，此时以“后进先出”的顺序（栈）移除等待的线程WaitNode节点
    // 2）若线程链表头节点为空，说明还没有线程调用Future.get()方法来获取任务执行结果，固然不用移除
    for (WaitNode q; (q = waiters) != null;) {
        // 调用UNSAFE的CAS方法将成员变量waiters设置为空
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                // 取出WaitNode节点的线程
                Thread t = q.thread;
                // 若取出的线程不为null，则将该WaitNode节点线程置空，且唤醒正在阻塞的该线程
                if (t != null) {
                    q.thread = null;
                    //【重要】唤醒正在阻塞的该线程
                    LockSupport.unpark(t);
                }
                // 继续取得下一个WaitNode线程节点
                WaitNode next = q.next;
                // 若没有下一个WaitNode线程节点，说明已经将所有等待的线程唤醒，此时跳出for循环
                if (next == null)
                    break;
                // 将已经移除的线程WaitNode节点的next指针置空，此时好被垃圾回收
                q.next = null; // unlink to help gc
                // 再把下一个WaitNode线程节点置为当前线程WaitNode头节点
                q = next;
            }
            break;
        }
    }
    // 不管任务正常执行还是抛出异常，都会调用done方法
    done();
    // 因为异步任务已经执行完且结果已经保存到outcome中，因此此时可以将callable对象置空了
    callable = null;        // to reduce footprint
}
```
`finishCompletion`方法的作用就是不管异步任务正常还是异常结束，此时都要唤醒且移除线程等待链表的等待线程节点，这个链表实现的是一个是`Treiber stack`，因此唤醒（移除）的顺序是"后进先出"即后面先来的线程先被先唤醒（移除），关于这个线程等待链表是如何成链的，后面再继续分析。

### 4.4.2 FutureTask的handlePossibleCancellationInterrupt方法
在`4.4小节`分析的`run`方法里的最后有一个`finally`块，此时若任务状态`state >= INTERRUPTING`，此时说明有其他线程执行了`cancel(true)`方法，此时需要让出`CPU`执行的时间片段给其他线程执行，我们来看下具体的源码：
```java
// FutureTask.java

private void handlePossibleCancellationInterrupt(int s) {
    // It is possible for our interrupter to stall before getting a
    // chance to interrupt us.  Let's spin-wait patiently.
    // 当任务状态是INTERRUPTING时，此时让出CPU执行的机会，让其他线程执行
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt

    // assert state == INTERRUPTED;

    // We want to clear any interrupt we may have received from
    // cancel(true).  However, it is permissible to use interrupts
    // as an independent mechanism for a task to communicate with
    // its caller, and there is no way to clear only the
    // cancellation interrupt.
    //
    // Thread.interrupted();
}
```
> **思考**： 为啥任务状态是`INTERRUPTING`时，此时就要让出CPU执行的时间片段呢？还有为什么要在义务任务执行后才调用`handlePossibleCancellationInterrupt`方法呢？

## 4.5 FutureTask.get方法,获取任务执行结果

```java
前面我们起一个线程在其`run`方法中执行异步任务后，此时我们可以调用`FutureTask.get`方法来获取异步任务执行的结果。

// FutureTask.java

public V get() throws InterruptedException, ExecutionException {
    int s = state;
    // 【1】若任务状态<=COMPLETING，说明任务正在执行过程中，此时可能正常结束，也可能遇到异常
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    // 【2】最后根据任务状态来返回任务执行结果，此时有三种情况：1）任务正常执行；2）任务执行异常；3）任务被取消
    return report(s);
}
```
可以看到，如果任务状态`state<=COMPLETING`，说明异步任务正在执行过程中，此时会调用`awaitDone`方法阻塞等待；当任务执行完后，此时再调用`report`方法来报告任务结果，此时有三种情况：1）任务正常执行；2）任务执行异常；3）任务被取消。

### 4.5.1 FutureTask.awaitDone方法
`FutureTask.awaitDone`方法会阻塞获取异步任务执行结果的当前线程，直到异步任务执行完成。
```java
// FutureTask.java

private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    // 计算超时结束时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    // 线程链表头节点
    WaitNode q = null;
    // 是否入队
    boolean queued = false;
    // 死循环
    for (;;) {
        // 如果当前获取任务执行结果的线程被中断，此时移除该线程WaitNode链表节点，并抛出InterruptedException
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        // 【5】如果任务状态>COMPLETING，此时返回任务执行结果，其中此时任务可能正常结束（NORMAL）,可能抛出异常（EXCEPTIONAL）
        // 或任务被取消（CANCELLED，INTERRUPTING或INTERRUPTED状态的一种）
        if (s > COMPLETING) {
            // 【问】此时将当前WaitNode节点的线程置空，其中在任务结束时也会调用finishCompletion将WaitNode节点的thread置空，
            // 这里为什么又要再调用一次q.thread = null;呢？
            // 【答】因为若很多线程来获取任务执行结果，在任务执行完的那一刻，此时获取任务的线程要么已经在线程等待链表中，要么
            // 此时还是一个孤立的WaitNode节点。在线程等待链表中的的所有WaitNode节点将由finishCompletion来移除（同时唤醒）所有
            // 等待的WaitNode节点，以便垃圾回收；而孤立的线程WaitNode节点此时还未阻塞，因此不需要被唤醒，此时只要把其属性置为
            // null，然后其有没有被谁引用，因此可以被GC。
            if (q != null)
                q.thread = null;
            // 【重要】返回任务执行结果
            return s;
        }
        // 【4】若任务状态为COMPLETING，此时说明任务正在执行过程中，此时获取任务结果的线程需让出CPU执行时间片段
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        // 【1】若当前线程还没有进入线程等待链表的WaitNode节点，此时新建一个WaitNode节点，并把当前线程赋值给WaitNode节点的thread属性
        else if (q == null)
            q = new WaitNode();
        // 【2】若当前线程等待节点还未入线程等待队列，此时加入到该线程等待队列的头部
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        // 若有超时设置，那么处理超时获取任务结果的逻辑
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        // 【3】若没有超时设置，此时直接阻塞当前线程
        else
            LockSupport.park(this);
    }
}
```
`FutureTask.awaitDone`方法主要做的事情总结如下：

0. 首先`awaitDone`方法里面是一个死循环；
1. 若获取结果的当前线程被其他线程中断，此时移除该线程WaitNode链表节点，并抛出InterruptedException；
2. 如果任务状态`state>COMPLETING`，此时返回任务执行结果；
3. 若任务状态为`COMPLETING`，此时获取任务结果的线程需让出CPU执行时间片段；
4. 若`q == null`，说明当前线程还未设置到`WaitNode`节点，此时新建`WaitNode`节点并设置其`thread`属性为当前线程；
5. 若`queued==false`，说明当前线程`WaitNode`节点还未加入线程等待链表，此时加入该链表的头部；
6. 当`timed`设置为true时，此时该方法具有超时功能，关于超时的逻辑这里不详细分析；
7. 当前面6个条件都不满足时，此时阻塞当前线程。

我们分析到这里，可以直到执行异步任务只能有一个线程来执行，而获取异步任务结果可以多线程来获取，当异步任务还未执行完时，此时获取异步任务结果的线程会加入线程等待链表中，然后调用调用`LockSupport.park(this);`方法阻塞当前线程。直到异步任务执行完成，此时会调用`finishCompletion`方法来唤醒并移除线程等待链表的每个`WaitNode`节点，这里这里唤醒（移除）`WaitNode`节点的线程是从链表头部开始的，前面我们也已经分析过。

还有一个特别需要注意的就是`awaitDone`方法里面是一个死循环，当一个获取异步任务的线程进来后可能会多次进入多个条件分支执行不同的业务逻辑，也可能只进入一个条件分支。下面分别举两种可能的情况进行说明：

**情况1**：
当获取异步任务结果的线程进来时，此时异步任务还未执行完即`state=NEW`且没有超时设置时：
1. **第一次循环**：此时`q = null`，此时进入上面代码标号`【1】`的判断分支，即为当前线程新建一个`WaitNode`节点；
2. **第二次循环**：此时`queued = false`，此时进入上面代码标号`【2】`的判断分支，即将之前新建的`WaitNode`节点加入线程等待链表中；
3. **第三次循环**：此时进入上面代码标号`【3】`的判断分支，即阻塞当前线程；
4.  **第四次循环**：加入此时异步任务已经执行完，此时进入上面代码标号`【5】`的判断分支，即返回异步任务执行结果。

**情况2**：
当获取异步任务结果的线程进来时，此时异步任务已经执行完即`state>COMPLETING`且没有超时设置时,此时直接进入上面代码标号`【5】`的判断分支，即直接返回异步任务执行结果即可，也不用加入线程等待链表了。
### 4.5.2 FutureTask.report方法
在`get`方法中，当异步任务执行结束后即不管异步任务正常还是异常结束，亦或是被`cancel`，此时获取异步任务结果的线程都会被唤醒，因此会继续执行`FutureTask.report`方法报告异步任务的执行情况，此时可能会返回结果，也可能会抛出异常。
```java
// FutureTask.java

private V report(int s) throws ExecutionException {
    // 将异步任务执行结果赋值给x，此时FutureTask的成员变量outcome要么保存着
    // 异步任务正常执行的结果，要么保存着异步任务执行过程中抛出的异常
    Object x = outcome;
    // 【1】若异步任务正常执行结束，此时返回异步任务执行结果即可
    if (s == NORMAL)
        return (V)x;
    // 【2】若异步任务执行过程中，其他线程执行过cancel方法，此时抛出CancellationException异常
    if (s >= CANCELLED)
        throw new CancellationException();
    // 【3】若异步任务执行过程中，抛出异常，此时将该异常转换成ExecutionException后，重新抛出。
    throw new ExecutionException((Throwable)x);
}
```
## 4.6 FutureTask.cancel方法,取消执行任务
我们最后再来看下`FutureTask.cancel`方法，我们一看到`FutureTask.cancel`方法，肯定一开始就天真的认为这是一个可以取消异步任务执行的方法，如果我们这样认为的话，只能说我们猜对了一半。
```java
// FutureTask.java

public boolean cancel(boolean mayInterruptIfRunning) {
    // 【1】判断当前任务状态，若state == NEW时根据mayInterruptIfRunning参数值给当前任务状态赋值为INTERRUPTING或CANCELLED
    // a）当任务状态不为NEW时，说明异步任务已经完成，或抛出异常，或已经被取消，此时直接返回false。
    // TODO 【问题】此时若state = COMPLETING呢？此时为何也直接返回false，而不能发出中断异步任务线程的中断信号呢？？
    // TODO 仅仅因为COMPLETING是一个瞬时态吗？？？
    // b）当前仅当任务状态为NEW时，此时若mayInterruptIfRunning为true，此时任务状态赋值为INTERRUPTING；否则赋值为CANCELLED。
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        // 【2】如果mayInterruptIfRunning为true，此时中断执行异步任务的线程runner（还记得执行异步任务时就把执行异步任务的线程就赋值给了runner成员变量吗）
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    // 中断执行异步任务的线程runner
                    t.interrupt();
            } finally { // final state
                // 最后任务状态赋值为INTERRUPTED
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    // 【3】不管mayInterruptIfRunning为true还是false，此时都要调用finishCompletion方法唤醒阻塞的获取异步任务结果的线程并移除线程等待链表节点
    } finally {
        finishCompletion();
    }
    // 返回true
    return true;
}
```
以上代码中，当异步任务状态`state != NEW`时，说明异步任务已经正常执行完或已经异常结束亦或已经被`cancel`，此时直接返回`false`;当异步任务状态`state = NEW`时，此时又根据`mayInterruptIfRunning`参数是否为`true`分为以下两种情况：

1. 当`mayInterruptIfRunning = false`时，此时任务状态`state`直接被赋值为`CANCELLED`，此时不会对执行异步任务的线程发出中断信号，**值得注意**的是这里对应的任务状态变化是**NEW -> CANCELLED**。
2. 当`mayInterruptIfRunning = true`时，此时会对执行异步任务的线程发出中断信号，**值得注意**的是这里对应的任务状态变化是**NEW -> INTERRUPTING -> INTERRUPTED**。

最后不管`mayInterruptIfRunning`为`true`还是`false`，此时都要调用`finishCompletion`方法唤醒阻塞的获取异步任务结果的线程并移除线程等待链表节点。

从`FutureTask.cancel`源码中我们可以得出答案，该方法并不能真正中断正在执行异步任务的线程，只能对执行异步任务的线程发出中断信号。如果执行异步任务的线程处于`sleep`、`wait`或`join`的状态中，此时会抛出`InterruptedException`异常，该线程可以被中断；此外，如果异步任务需要在`while`循环执行的话，此时可以结合以下代码来结束异步任务线程，即执行异步任务的线程被中断时，此时`Thread.currentThread().isInterrupted()`返回`true`，不满足`while`循环条件因此退出循环，结束异步任务执行线程，如下代码：
```
public Integer call() throws Exception {
    while (!Thread.currentThread().isInterrupted()) {
        // 业务逻辑代码
        System.out.println("running...");

    }
    return 666;
}
```

**注意**：调用了`FutureTask.cancel`方法，只要返回结果是`true`，假如异步任务线程虽然不能被中断，即使异步任务线程正常执行完毕，返回了执行结果，此时调用`FutureTask.get`方法也不能够获取异步任务执行结果，此时会抛出`CancellationException`异常。请问知道这是为什么吗？

因为调用了`FutureTask.cancel`方法，只要返回结果是`true`，此时的任务状态为`CANCELLED`或`INTERRUPTED`,同时必然会执行`finishCompletion`方法，而`finishCompletion`方法会唤醒获取异步任务结果的线程等待列表的线程，而获取异步任务结果的线程唤醒后发现状态`s >= CANCELLED`，此时就会抛出`CancellationException`异常了。

# 5 总结

好了，本篇文章对`FutureTask`的源码分析就到此结束了，下面我们再总结下`FutureTask`的实现逻辑：
1. 我们实现`Callable`接口，在覆写的`call`方法中定义需要执行的业务逻辑；
2. 然后把我们实现的`Callable`接口实现对象传给`FutureTask`，然后`FutureTask`作为异步任务提交给线程执行；
3. 最重要的是`FutureTask`内部维护了一个状态`state`，任何操作（异步任务正常结束与否还是被取消）都是围绕着这个状态进行，并随时更新`state`任务的状态；
4. 只能有一个线程执行异步任务，当异步任务执行结束后，此时可能正常结束，异常结束或被取消。
5. 可以多个线程并发获取异步任务执行结果，当异步任务还未执行完，此时获取异步任务的线程将加入线程等待列表进行等待；
6. 当异步任务线程执行结束后，此时会唤醒获取异步任务执行结果的线程，注意唤醒顺序是"后进先出"即后面加入的阻塞线程先被唤醒。
7. 当我们调用`FutureTask.cancel`方法时并不能真正停止执行异步任务的线程，只是发出中断线程的信号。但是只要`cancel`方法返回`true`，此时即使异步任务能正常执行完，此时我们调用`get`方法获取结果时依然会抛出`CancellationException`异常。

 > **扩展**： 前面我们提到了`FutureTask`的`runner`,`waiters`和`state`都是用`volatile`关键字修饰，说明这三个变量都是多线程共享的对象（成员变量），会被多线程操作，此时用`volatile`关键字修饰是为了一个线程操作`volatile`属性变量值后，能够及时对其他线程可见。此时多线程操作成员变量仅仅用了`volatile`关键字仍然会有线程安全问题的，而此时Doug Lea老爷子没有引入任何线程锁，而是采用了`Unsafe`的`CAS`方法来代替锁操作，确保线程安全性。

# 6 分析FutureTask源码，我们能学到什么？

我们分析源码的目的是什么？除了弄懂`FutureTask`的内部实现原理外，我们还要借鉴大佬写写框架源码的各种技巧，只有这样，我们才能成长。

分析了`FutureTask`源码，我们可以从中学到：
1. 利用`LockSupport`来实现线程的阻塞\唤醒机制；
2. 利用`volatile`和`UNSAFE`的`CAS`方法来实现线程共享变量的无锁化操作；
3. 若要编写超时异常的逻辑可以参考`FutureTask`的`get(long timeout, TimeUnit unit)`的实现逻辑；
4. 多线程获取某一成员变量结果时若需要等待时的线程等待链表的逻辑实现；
5. 某一异步任务在某一时刻只能由单一线程执行的逻辑实现；
6. `FutureTask`中的任务状态`satate`的变化处理的逻辑实现。
7. ...

以上列举的几点都是我们可以学习参考的地方。


**若您觉得不错，请无情的转发和点赞吧！**

【源码笔记】Github地址：

https://github.com/yuanmabiji/Java-SourceCode-Blogs

-------------------------------------------------------------------------------
公众号【源码笔记】，专注于Java后端系列框架的源码分析。

![](https://user-gold-cdn.xitu.io/2020/6/26/172ec7a63ebdf26f?w=498&h=143&f=png&s=23420)