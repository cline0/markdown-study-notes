`ThreadPoolExecutor` （TPE）用于创建自定义线程池。

其中，有 7 个自定义核心参数：

```java
public ThreadPoolExecutor(int corePoolSize,//线程池的核心线程数量
                          	//线程池的最大线程数
                            int maximumPoolSize,
                          	//当线程数大于核心线程数时，多余的空闲线程存活的最长时间
                            long keepAliveTime,
                           	//时间单位   
                          	TimeUnit unit,
                          	//任务队列，用来储存等待执行任务的队列
                            BlockingQueue<Runnable> workQueue,
                          	//线程工厂，用来创建线程，一般默认即可    
                          	ThreadFactory threadFactory,
                          	//拒绝策略，当提交的任务过多而不能及时处理时，我们可以定制策略来处理任务
                             RejectedExecutionHandler handler
```

线程池的核心线程默认不会被回收。

可以通过 `allowCoreThreadTimeOut(boolean value)` 方法来设置为 true，这样，当核心线程超过了 `keepAliveTime` 的时候，就会被回收。

更多内容可以看《12306 项目学习 -> 组件库 -> 公共组件库》。

动态线程池是指在运行时动态修改线程池的核心线程数、最大线程数和使用的工作队列。

动态线程池的工作原理是使用 Nacos 等配置中心，管理参数配置。

Nacos 等配置中心会将配置的修改推送到应用，应用调用相应的 `setter` 进行设置。

  