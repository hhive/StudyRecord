### volatile

#### 功能 ####

1、保证变量线程间的可见性

2、禁止指令重排序

#### 实现原理

从三个层级讨论

1、字节码

> ACC_VOLATILE

2、jvm

在变量操作之前加上内存屏障，注意，jvm的内存屏障不同于处理器的内存屏障

> StoreStoreBarrier
>
> volatile 写操作
>
> StoreLoadBarrier

> LoadLoadBarrier
>
> volatile 读操作
>
> LoadStoreBarrier

3、处理器

> lock指令

### synchronized



#### 实现原理

1、字节码

> monitor enter
>
> monitor exit

2、jvm

C++的锁实现

3、处理器

lock comxchg



### ThreadLocal

```java
public class ThreadLocalTest {
    //ThreadLocal可以只使用一个变量而在每一个线程保存一份线程独有的数据
    private static final ThreadLocal<String> sThreadLocal = new ThreadLocal<>();
    //可获取线程创建之前主线程数据的InheritableThreadLocal,但注意，一个ThreadLocal在每个线程都只能
    //保存一个值，因此如果子线程又调用了set，会覆盖主线程的值
    public static ThreadLocal<String> iThreadLocal = new InheritableThreadLocal<String>();

    private void showThreadLocal() {
        sThreadLocal.set("showThreadLocal: 这是在主线程中");
        System.out.println("线程名字：" + Thread.currentThread()
            .getName() + "---" + sThreadLocal.get());
        //线程a
        new Thread(new Runnable() {
            @Override
            public void run() {
                sThreadLocal.set("这是在线程a中");
                System.out.println("线程名字：" + Thread.currentThread()
                    .getName() + "---" + sThreadLocal.get());
            }
        }, "线程a").start();
    }

    private void showInheritableThreadLocal() {
        iThreadLocal.set("showInheritableThreadLocal: 这是在主线程中");
        System.out.println("线程名字：" + Thread.currentThread()
            .getName() + "---" + iThreadLocal.get());
        //线程b
        new Thread(new Runnable() {
            @Override
            public void run() {
                // iThreadLocal.set("这是在线程b中");
                System.out.println("线程名字：" + Thread.currentThread()
                    .getName() + "---" + iThreadLocal.get());
            }
        }, "线程b").start();
        //线程c，如果子线程重新设值，会替代主线程的值，因为存入map的key是同一个
        new Thread(() -> {
            iThreadLocal.set("这是在线程c中");
            System.out.println("线程名字：" + Thread.currentThread()
                .getName() + "---" + iThreadLocal.get());
        }, "线程c").start();
    }
    public static void main(String[] args) {
        ThreadLocalTest threadLocalTest = new ThreadLocalTest();
        threadLocalTest.showThreadLocal();
        threadLocalTest.showInheritableThreadLocal();

    }
}

线程名字：main---showThreadLocal: 这是在主线程中
线程名字：main---showInheritableThreadLocal: 这是在主线程中
线程名字：线程a---这是在线程a中
线程名字：线程b---showInheritableThreadLocal: 这是在主线程中
线程名字：线程c---这是在线程c中
```

#### ThreadLocal:

![ThreadLocal存取流程](JUC.assets/ThreadLocal存取流程-16384151560131.jpg)

如main和线程演示，主线程中创建ThreadLocal，每个子线程（Thread）都有一个自己的ThreadLocal.ThreadLocalMap变量，map以主线程的ThreadLocal作为key保存一个数据，这样取值的时候都能通过主线程的ThreadLocal这一唯一的key取到自己线程独有的数据，主线程有多个ThreadLocal，就可以存取多个数据

#### InheritableThreadLocal

InheritableThreadLocal继承自ThreadLocal，如线程b所示，是可继承主线程数据的ThreadLocal。

```java
private Thread(ThreadGroup g, Runnable target, String name,
               long stackSize, AccessControlContext acc,
               boolean inheritThreadLocals) {
    ...
    
    Thread parent = currentThread();
    ....
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        //复制主线程的数据到子线程
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
	...

    /* Set thread ID */
    this.tid = nextThreadID();
}

inheritableThreadLocals是Thread一个变量
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

在调用Thread构造方法时（new Thread是在主线程完成的），如果主线程inheritableThreadLocals不为空，则将主线程inheritableThreadLocals中的值复制给子线程。

```java
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }
```

```java
//如果主线程的ThreadLocalMap有值，则复制到子线程的ThreadLocalMap
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (Entry e : parentTable) {
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                //可以继承InheritableThreadLocal，重写childValue根据需要获取数据
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

InheritableThreadLocal的存取复用ThreadLocal的方法



#### ThreadLocalMap的key为什么继承弱引用

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

如果一个对象**仅被**一个弱引用指向，那么当下一次GC到来时，这个对象一定会被垃圾回收器回收掉。

这里模拟一种场景，主线程新建Threadlocal对象，赋值“hello world main”，子线程也利用这个Threadlocal对象赋值“new thread”，此时内存引用情况如下：

![img](JUC.assets/v2-0a7df13e262f4a61ebba7fc20ae96836_1440w.jpg)

假设两条红线是强引用，此时使得threadLocal = null;

![img](JUC.assets/v2-52355ad50ad93d41d7703cad682e9c40_1440w.jpg)

​		虽然两个线程都主动释放掉了对ThreadLocal对象的引用，但是，从主线程thread引用->ThreadLocal对象，依然存在这一条可达路径。众所周知，现今主流JVM判断一个对象是否可回收的算法通常为可达路径算法，而不是引用[计数法](https://www.zhihu.com/search?q=计数法&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A139214244})。可达路径算法以GCROOT出发，如果存在一条通向某个对象的强引用通路，那么这个对象是永远不会回收掉的(即便发生OOM也不会回收)。thread的引用是主线程的一个[本地变量](https://www.zhihu.com/search?q=本地变量&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A139214244})，根据GCROOT算法，thread的引用是可以作为一个GCROOT的，那么现状就是：我们显式地释放掉了threadLocal的引用(threadLocal = null;)，因为我们确认后续我们不会使用到它了，但是，由于存在GCROOT的一条可达通路，程序并没有像我们希望的那样立刻释放掉ThreadLocal对象，直到我们所有的线程都释放掉了，即程序结束，ThreadLocal对象才会被真正的释放掉，这无疑就是内存泄露。为了解决这个问题，我们把图中的红线换成弱引用，如下图所示：

![img](JUC.assets/v2-2081ed02d4874866c018192bdee92249_1440w.jpg)

































### 1

