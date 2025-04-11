`ForkJoinPool` 适用于 CPU 密集型任务，不适用于 IO 密集型任务。

`ForkJoinPool` 的核心是将任务单个大任务拆分成多个小任务，然后单独计算多个小任务的结果，并将其合并。

拆分与合并的逻辑是由自己编写的。

所以，`ForkJoinPool` 的核心是`分治算法`。

分治算法的一般步骤：

1. 分解任务：将要解决的问题划分为若干规模更小的同类问题
2. 求解任务：当子问题划分得足够小时，开始进行计算
3. 合并结果：将求解的结果合并为原问题的结果

## ForkJoinTask

任务由 `ForkJoinTask` 抽象类来表示（承载），它有 3 个子类：

- `RecursiveTask`：有递的过程，也有归的过程

- `RecursiveAction`：只有递的过程
- `CountedCompleter`：

| 子类                  | 是否有返回值 | 适用场景                                 | 典型示例                         |
| --------------------- | ------------ | ---------------------------------------- | -------------------------------- |
| `RecursiveTask<V>`    | ✅ 有返回值   | 计算任务、递归分治                       | **数组求和**、归并排序、矩阵运算 |
| `RecursiveAction`     | ❌ 无返回值   | 数据修改、并行遍历                       | **批量更新数组**、图像处理       |
| `CountedCompleter<V>` | ✅ 灵活控制   | 多任务间有依赖的情况，支持任务完成后回调 | **任务调度**、图遍历             |

上面的 3 个子类都是抽象类，都需要子类至少实现 `compute()` 方法才能够使用。

分解任务和合并结果都是在 `compute` 方法中完成的。

`ForkJoinTask` 有 2 个核心方法 `fork()` 和 `join()`，分别对应任务的分解和合并：

- `fork()`：由子任务调用，可以向当前原任务所在的线程池提交该子任务
- `join()`：由子任务调用，可以获取该子任务的执行结果

所以，我们要单独地创建子任务，才能够实现分解任务。

## ForkJoinPool

ForkJoinPool 有 4 个核心参数：

- `parallelism: int`：指定并行级别（parallelism level），即工作线程的数量。如果未设置的话，将设置为处理的数量
- `factory: ForkJoinWorkerThreadFactory `：用于创建工作线程的工厂

- `handler: UncaughtExceptionHandler `:指定异常处理器，当任务在运行中出错时，将由设定的处理器处理

- `asyncMode: boolean `：设置队列的工作模式：`asyncMode ? FIFO_QUEUE : LIFO_QUEUE` 

可以有 3 种方法用于向 `ForkJoinPool` 提价任务：

| 方法                        | 返回值            | 是否阻塞调用线程         | 适用场景                                        |
| --------------------------- | ----------------- | ------------------------ | ----------------------------------------------- |
| `execute(Runnable)`         | `void`            | ❌ **不阻塞**             | 适用于无需获取结果的任务                        |
| `execute(ForkJoinTask<?>)`  | `void`            | ❌ **不阻塞**             | 适用于不关心返回值的 ForkJoinTask               |
| `submit(Runnable/Callable)` | `ForkJoinTask<T>` | ❌ **不阻塞**             | 适用于需要异步获取结果的任务                    |
| `submit(ForkJoinTask<T>)`   | `ForkJoinTask<T>` | ❌ **不阻塞**             | 适用于 ForkJoinTask，后续可用 `join()` 获取结果 |
| `invoke(ForkJoinTask<T>)`   | `T`（任务的结果） | ✅ **阻塞，直到任务完成** | 适用于需要立即获取计算结果的任务                |

注意到，其中可以运行 `Runalbe` 或者 `Callable` 的任务，但是此类任务无法分解，只能提供任务窃取的好处。

### 工作队列和任务窃取

任务窃取是指：空闲线程从繁忙线程的双端工作队列中窃取一个任务，然后执行。

默认情况下，工作线程从它自己的双端队列的**头部**获取任务。

当自己的任务为空时，线程会从其他繁忙线程双端队列的**尾部**中获取任务。这种方法，最大限度地减少了线程竞争任务的可能性。

这个双端工作队列有如下特性：

- 由内部类 `WorkQueue` 实现
- 仅支持 `push`、`pop` 和 `poll`
- `push` 和 `pop` 只能由工作队列所属的线程调用
- `poll` 可以被其它线程调用，用与其它线程窃取任务

`fork()` 会将任务提交到工作队列的尾部。

## 示例代码

计算 1 到 n 的累加和：

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

public class ForkJoinSumExample {
    // 任务类，继承 RecursiveTask
    static class SumTask extends RecursiveTask<Long> {
        private final int start, end;
        private static final int THRESHOLD = 10; // 阈值

        public SumTask(int start, int end) {
            this.start = start;
            this.end = end;
        }

        @Override
        protected Long compute() {
            // 任务足够小，直接计算
            if ((end - start) <= THRESHOLD) {
                long sum = 0;
                for (int i = start; i <= end; i++) {
                    sum += i;
                }
                return sum;
            } 
            // 任务过大，拆分
            else {
                int mid = (start + end) / 2;
                SumTask leftTask = new SumTask(start, mid);
                SumTask rightTask = new SumTask(mid + 1, end);

                leftTask.fork();  // 异步执行子任务
                long rightResult = rightTask.compute(); // 直接计算右任务
                long leftResult = leftTask.join(); // 等待左任务结果
                
                return leftResult + rightResult;
            }
        }
    }

    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();
        SumTask task = new SumTask(1, 100);
        long result = pool.invoke(task);
        System.out.println("累加结果: " + result);
    }
}
```

