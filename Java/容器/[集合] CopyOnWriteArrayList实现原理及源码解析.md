CopyOnWriteArrayList是Java并发包里提供的并发类，简单来说它就是一个线程安全且读操作无锁的ArrayList。正如其名字一样，在写操作时会复制一份新的List，在新的List上完成写操作，然后再将原引用指向新的List。这样就保证了写操作的线程安全。这是一种*读写分离*的并发策略。类似的还有CopyOnWrite

### 实现原理
我们都知道，集合框架中的ArrayList是非线程安全的，Vector虽是线程安全的，但由于简单粗暴的锁同步机制，性能较差。而CopyOnWriteArrayList则提供了另一种不同的并发处理策略。针对*读多写少*的并发场景，CopyOnWriteArrayList允许线程并发访问读操作，这个时候是没有加锁限制的，性能较高。而写操作的时候，则首先将容器复制一份，然后在新的副本上执行写操作，这个时候写操作是上锁的。结束之后再将原容器的引用指向新容器。注意，在上锁执行写操作的过程中，如果有需要读操作，会作用在原容器上。因此上锁的写操作不会影响到并发访问的读操作。

了解了实现原理，我们也能很容易总结出容器的优缺点。优点是读操作性能很高，无需任何同步措施，适用于读多写少的并发场景。缺点一是需要额外内存占用，毕竟每次写操作需要复制一份，数据量大时会对内存压力较大，可能会引起频繁的GC。二是无法保证强一致性，毕竟在复制写操作过程中，读和写分别作用在新老不同容器上。读操作虽然不会阻塞，但在这段时间内读取的还是老容器的数据。

### 源码分析
这里还是通过贴代码加注释的方式来解析。

首先，存储数据的结构是array, 为了保持可见性加了volatile关键字。
```
/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;
```

*add方法*
```
public boolean add(E e) {
    // 写方法，需要先ReentrantLock锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 将原数组复制一份, 长度+1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 在新的副本上执行添加操作
        newElements[len] = e;
        // 将原容器引用指向新的副本
        setArray(newElements);
        return true;
    } finally {
        // 解锁
        lock.unlock();
    }
}
```

*remove方法*
```
public E remove(int index) {
    // 同样用ReentrantLock加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            // 如果要删除的是末端数据，则只用复制前面len-1个数据到新副本上，并切换引用
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            // 如果不在末端，则分两次复制其它元素到新副本上，并切换引用
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                                numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

*get方法*
```
public E get(int index) {
    return get(getArray(), index);
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```
直接从数组中取即可，无需加锁。

### 小结
本文分析了CopyOnWriteArrayList的实现原理和源码进行分析，并总结了优缺点。实际上来说，并发场景有很多，针对不同的并发场景适用不同的优化策略（比如这里的读多写少场景），平时写代码时，还是需要根据特定场景对并发容器做出合理的选择和应用。