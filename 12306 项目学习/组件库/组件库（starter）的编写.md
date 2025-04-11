所谓的组件库就是编写一个库供其他的库调用。类似于 rust 的 mod。

下面所讲的组件库是基于 Spring-Boot 来完成的。所以，每个库相当于一个 Spring-Boot 项目。

## 步骤
### 步骤一
创建一个 SpringBoot 项目，将库所提供的 Bean 注册好。

```java
public class HelloWorld {
    public void sayHello() {
        System.out.println("hello world");
    }
}
```

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class Config {
    @Bean
    public HelloWorld getBean() {
        return new HelloWorld();
    }
}
```

### 步骤二
在 `resources` 目录下创建 `META-INF` 目录。

如果是 SpringBoot 2.x，则在 `META-INF` 目录中添加文件 `spring.factories`，该文件的内容为：

```latex
org.springframework.boot.autoconfigure.EnableAutoConfiguration=<Config 类的完全限定名>
```

如果是 SpringBoot 3.x，则在 `META-INF` 目录中添加文件 `org.springframework.boot.autoconfigure.AutoConfiguration.imports`，其中的内容为：

```latex
<Config 类的完全限定名>
```

### 步骤三
在要使用该库的项目中，通过 `pom.xml` 引入改库，则改库中定义的 Bean 就可以被使用了。

## 可插拔组件库
所谓的**可插拔**是指可以通过条件判断来只加载部分 Bean、或者配置类（配置类中定义了 Bean）。

库可以进行可插拔是由库的编写者来决定的，而不是库的使用者来决定的。

条件判断一般通过命名为 `@ConditionalOn<xxx>` 的注解来完成。

| 注解名称 | 描述 | 条件成立时的行为 |
| --- | --- | --- |
| `@ConditionalOnBean` | 根据容器中是否存在指定的 `Bean` 来决定是否加载配置或 `Bean`。 | 如果容器中存在指定类型或名称的 `Bean`，则条件成立，相关的配置或 `Bean` 会被加载。 |
| `@ConditionalOnMissingBean` | 与 `@ConditionalOnBean` 相反，当容器中**没有**某个 `Bean` 时，才加载配置或 `Bean`。 | 只有当容器中没有某个 `Bean` 时，条件成立，相关配置或 `Bean` 会被加载。 |
| `@ConditionalOnClass` | 根据类路径中是否存在某些类来决定是否加载配置或 `Bean`。 | 如果类路径中存在指定的类，条件成立，相关配置或 `Bean` 会被加载。 |
| `@ConditionalOnMissingClass` | 与 `@ConditionalOnClass` 相反，判断类路径中**不存在**某些类时，才加载配置或 `Bean`。 | 如果类路径中没有某个指定的类，条件成立，相关配置或 `Bean` 会被加载。 |
| `@ConditionalOnProperty` | 根据指定的属性是否存在或是否满足某个条件（如值）来决定是否加载配置或 `Bean`。 | 如果属性存在且满足条件（如值为 `true` 或某个特定值），则条件成立，相关配置或 `Bean` 会被加载。 |
| `@ConditionalOnExpression` | 根据 Spring EL 表达式的值来决定是否加载配置或 `Bean`。 | 如果表达式结果为 `true`，则条件成立，相关配置或 `Bean` 会被加载。 |
| `@ConditionalOnResource` | 根据类路径中是否存在特定资源（文件、路径等）来决定是否加载配置或 `Bean`。 | 如果指定的资源存在，条件成立，相关配置或 `Bean` 会被加载。 |
| `@ConditionalOnJava` | 根据 JVM 的版本来决定是否加载配置或 `Bean`。 | 如果当前 Java 版本符合指定条件，相关配置或 `Bean` 会被加载。 |


### 例子（`@ConditionalOnBean`）
在库中，**库的编写者**定义一个用于标记的注解：

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface EnableHelloWorld {
}
```

然后，在定义配置类 `Config` 的时候，使用 `@ConditionalOnBean` 注解：

```java
@Configuration
@ConditionalOnBean(annotation = EnableHelloWorld.class)
public class Config {
    @Bean
    public HelloWorld getBean() {
        return new HelloWorld();
    }
}
```

然后由**库的使用者**使用 `@EnableHelloWorld` 来控制是否加载 `Config` 配置类中定义的 Bean：

```java
@EnableHelloWorld
@SpringBootApplication
public class MultiComponentApplication {
    public static void main(String[] args) {
        SpringApplication.run(MultiComponentApplication.class, args);
    }
}
```

如果 `Application` 没有 `@EnableHelloWorld` 注解，则无法在其它地方使用 `HelloWorld` Bean。

