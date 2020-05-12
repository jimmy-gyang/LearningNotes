在并发场景中，我们常常需要防止任务在共享资源上产生冲突。其中有一个解决思路是根除对变量的共享。线程本地存储就是这样的一种自动化机制，可以为使用相同变量的每个不同的线程都创建不同的存储。因此，如果你有5个线程都要使用变量x所表示的对象，那线程本地存储就会生成5个用于x的不同的存储块。主要是他们使得你可以将状态与线程关联起来。

创建和管理线程本地存储就是由ThreadLocal类来实现的。ThreadLocal一般称为线程本地变量，它是一种特殊的线程绑定机制，将变量与线程绑在一起，为每一个线程维护一个独立的变量副本。变量的可见范围限制在同一个线程内。

### 示例代码
这里开启五条线程，每一条线程会通过ThreadLocal绑定一个id数据，从0到5。由于会发生竞争，因此使用AtomicInteger原子类来完成自增操作。
```
public class ThreadLocalTest {
    static class ThreadId {
        public static AtomicInteger id = new AtomicInteger(0);

        public static final ThreadLocal<Integer> threadIds = new ThreadLocal<Integer>() {
            @Override
            protected Integer initialValue() {
                return id.getAndIncrement();
            }
        };

        public static int getId() {
            return threadIds.get();
        }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + ": " + ThreadId.getId());
            }).start();
        }
    }
}
```
输出结果如下, 每个线程都拥有了独立的变量id。
```
Thread-0: 0
Thread-1: 1
Thread-2: 2
Thread-3: 3
Thread-4: 4
```

### 源码
这里展示ThreadLocal的set和get方法，看看它是如何为线程绑定变量的。这里依然通过注释的方式来分析代码。

*set*方法
```
public void set(T value) {
    // 1. 获取当前对象
    Thread t = Thread.currentThread();
    // 2. 获取该线程对象的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 如果map不为空，则执行set操作。当前threadLocal对象为key，实际存储对象为value。
        map.set(this, value);
    else
        // 如果map为空，则创建该线程的ThreadLocalMap
        createMap(t, value);
}
```

ThreadLocalMap保留了线程独有的变量，由ThreadLocal类来维护。
```
/* ThreadLocal values pertaining to this thread. This map is maintained
    * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

*get*方法
```
public T get() {
    // 1. 获取当前对象
    Thread t = Thread.currentThread();
    // 2. 获取该线程对象的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 3.如果map不为空，以threadlocal实例为key获取到对应Entry，然后从Entry中取出对象
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 4. 如果map为空就调用了此get()方法, 则初始化ThreadLocalMap，并设置初始值为null
    return setInitialValue();
}
```

看到这里应该能明白了，线程独享变量存储于ThreadLocalMap里，而ThreadLocal其实就是用来提供访问的类。

### 小结
ThreadLocal让每个单独的线程都分配了自己的存储。但是注意，它只是给开发者们提供了一个解决共享对象访问的方法。并不是意味着各个线程之间做到了完全的独享。要实现线程彻底的独享变量，意味着每个线程都得拥有一套新的**栈中的对象引用+堆中的对象**，显然是一件耗资巨大的事情。而普遍的情况是，所谓的独享变量副本，其实也就是每个线程都拥有一个独立的对象引用，而堆中的对象还是线程间共享的，因此依然会有线程不安全的风险。

所以，需不需要完全独享变量进行完全的隔离，那就看具体的应用场景了。ThreadLocal在spring的事务管理，web开发中的http session管理中都有出现, 这种一个请求对应一个线程的场景就很适合使用ThreadLocal。我们平时在应用中也应当注意使用ThreadLocal，避免产生并发问题。