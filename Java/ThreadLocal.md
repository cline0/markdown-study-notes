`ThreadLocal` 位于 `Thread`  类的 ` ThreadLocal.ThreadLocalMap threadLocals;` 中。

从中可以看出，`ThreadLocal` 使用了 `ThreadLocalMap` 这一内部类来存储多个 `ThreadLocal`，下面来看一看 `ThreadLocaMap` 如何存储多个 `ThreadLocal`：

```java
static class ThreadLocalMap {
	// 使用 **弱引用** 来关联 Entry
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            // 这里调用 super 创建了一个指向 ThreadLocal 的**弱引用**
            // 相当于 Entry 还有一个变量，该变量的类型为 ThreadLocal，但是是一个弱引用，而不是一个强引用
            super(k);
            value = v;
        }
    }
    
    // 使用一个 Entry 数组来存储 ThreadLocal 的 value
    private Entry[] table;
}
```

其中，`table` 字段存储了该线程的所有 ThreadLocal 的变量。

> Entry 中的 key（即 `ThreadLocal` 实例）为什么是弱引用的？
>
> 下面是几点事实：
>
> - 当我们创建一个 `ThreadLocal` 的时候，是一个强引用。
> - 当包含 `ThreadLocal` 的类被回收的时候，此强引用也会消失
>
> 如果在 `ThreadLocalMap` 中使用强引用，由于 `ThreadLocalMap` 是位于线程的局部变量 `Thread` 中的，如果该 `Thread` 的生命周期比该类的生命周期长，而 `ThreadLocalMap` 中还有该 ThreadLocal 的引用，则会导致该对象不会被回收，从而导致**内存泄漏**。
>
> 而使用弱引用就会使得类中的强引用被删除的时候，其中的 `ThreadLocal` 一定会被回收（即使没有手动调用）。

当强引用消失被垃圾回收之后， 弱引用中 `Threadlocal` 对应的值为 `null`，所以，如果当 `ThreadLocalMap` 中的方法 `set()`、`getEntry()` 和 `remove()` 方法中，如果检测到该 key 为 null，则会将该 Entry 中的 value 和该 Entry 也设置为 null，让垃圾回收期回收。同时如果 key 为 null，则也会遍历后面的 Entry 以清理其它 key 为 null 的 value。

这样的话，就会使得对应的 Entry 被 GC 回收。

但是，前提是同一线程的其它 `ThreadLocal` 要调用 `set`、`get` 或者 `remove` 方法并且对应的 key 为 null，否则，因为 `ThreadLocalMap` 没有定期任务扫描 key 是否为 null，所以，Entry 中的 value 会导致存在**内存泄漏**的风险。

所以，在实践中，**要确保手动调用 `ThreadLocal` 的 `remove` 方法，以手动清除 value**。

同时，为了能够在类的声明周期中始终能够手动清除 value，还需要将 `ThreadLocal` 变量声明为 static。如果不声明为 static，则对应的实例 GC 之后，无法手动清除 value 了。

> 当调用 `ThreadLocal` 的 `get()` 方法的时候，如何确定该 `ThreadLocal` （即上文所说的 `ThreadLocalMap` 中的 key ）在哪里呢？
>
> 答案是每一个 `ThreadLocal` 都有一个 `threadLocalHashCode` 字段，该字段使用一个哈希函数（一个线程中的 `AtomicInteger` 增加一个固定的值来生成的）来生成一个哈希值，然后对该值取模从而得到位于 `table` 中的索引。
>
> 如果出现了哈希冲突，则使用**线性探测法**来找到下一个位置。

所以，通过上面的分析，**内存泄漏**的原因是：

- 在类实例的生命周期结束之前，没有手动调用 `remove` 方法清理掉 `ThreadLocal` 中的 value

## TransmittableThreadLocol (TTL)

TTL 主要解决 2 个问题：

- 子线程可以访问父线程的 `ThreadLocal` 值
- 如果旧任务的线程被复用（线程池中的线程），则新的任务访问到旧任务的 `ThreadLocal` 数据

### 实现原理

TTL 继承了 `InheritableThreadLocal`，以实现**在创建子线程**时，会将父线程中的 `ThreadLocal` 值复制到子线程中。

但是 `InheritableThreadLocal` 的局限性是只能在创建子线程的时候才会复制，在线程池复用的场景下就不可行了。

为此，TTL 选择了在任务提交的时候进行 `ThreadLocal` 的复制，实现原理是：

- 自定义自己的 `TtlRunable` 或者 `TtlCallable`，并使其实现 `Runalbe` 或者 `Callable` 接口
- 这样的话，相当于实现了一个 AOP，则真正运行 `Runable` 或者 `Callable` 之前，设置子线程的 `TreadLocal` 值

下面是简化后的源码（不是真实的源码）：

```java
public final class TtlRunnable implements Runnable {
    private final Runnable runnable;
    private final Map<TransmittableThreadLocal<?>, Object> captured;

    private TtlRunnable(Runnable runnable) {
        this.runnable = runnable;
        this.captured = TransmittableThreadLocal.COPY_TTL_VALUES();
    }

    public static TtlRunnable get(Runnable runnable) {
        return runnable == null ? null : new TtlRunnable(runnable);
    }

    @Override
    public void run() {
        Map<TransmittableThreadLocal<?>, Object> backup = TransmittableThreadLocal.RESTORE_TTL_VALUES(captured);
        try {
            runnable.run(); // 执行真正的任务
        } finally {
            TransmittableThreadLocal.RESTORE_TTL_VALUES(backup); // 恢复现场，防止污染
        }
    }
}
```

如果想要对现有代码透明地使用 TTL，可以使用 `Java agent` 来完成。

## FastThreadLocal

FastThreadLocal 主要对 `ThreadLocal` 的改进在于不使用哈希函数来获取 Entry 在数组中的位置，而是使用   一个 `AtomicInteger` 类型的 `index` 变量来生成，每有一个 `ThreadLocal`，其值就加一。

其中，`FastThreadLocal` 类中多增加了一个该 `ThreadLocal` 的索引，所以，可以直接在 $O(1)$ 的时间内查找到对应的 value。

