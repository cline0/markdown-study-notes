Java Agent 是 JVM 的一个命令参数，用于指定一个 JAR 包，该 JAR 包可以在不修改源代码的情况下动态改变类的行为或采集运行时数据。

其中，JVM 中与 Java Agent 有关的参数有：

```bash
-agentlib:<libname>[=<选项>] 加载本机代理库 <libname>, 例如 -agentlib:hprof
	参阅 -agentlib:jdwp=help 和 -agentlib:hprof=help
-agentpath:<pathname>[=<选项>]
	按完整路径名加载本机代理库
-javaagent:<jarpath>[=<选项>]
	加载 Java 编程语言代理, 请参阅 java.lang.instrument
```

> **本机代理库**
>
> 相对于 Java 语言编写的 Java Agent（基于 Instrumentation API），本机代理库直接与 JVM 交互，能够获取更底层的信息，如垃圾回收（GC）、线程、类加载、JIT 编译、内存分配等，因此适用于更高级的 JVM 监控工具和性能分析工具。
>
> 本机代理库通常由 C/C++ 代码编写，然后编译成共享库（Linux 下的 `.so` 文件，Windows 下的 `.dll` 文件，macOS 下的 `.dylib` 文件），通过 `-agentpath` 或 `-agentlib` 选项加载到 JVM。

TODO：https://www.cnblogs.com/rickiyang/p/11368932.html