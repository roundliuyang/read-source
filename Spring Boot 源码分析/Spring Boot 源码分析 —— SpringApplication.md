## 1. 概述

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication // <1>
public class MVCApplication {

    public static void main(String[] args) {
        SpringApplication.run(MVCApplication.class, args); // <2>
    }

}
```

- `<1>` 处，使用 `@SpringBootApplication` 注解，标明是 Spring Boot 应用。通过它，可以开启自动配置的功能。
- `<2>` 处，调用 `SpringApplication#run(Class<?>... primarySources)` 方法，启动 Spring Boot 应用。

上述的代码，是我们使用SpringBoot  时，最最最常用的代码。而本文，我们先来分析 Spring Boot 应用的**启动过程**。



## 2. SpringApplication

`org.springframework.boot.SpringApplication` ，Spring 应用启动器。正如其代码上所添加的注释，它来提供启动 Spring 应用的功能。

```
`Class that can be used to bootstrap and launch a Spring application from a Java main method.`
```

大多数情况下，我们都是使用它提供的**静态**方法：

```java
// SpringApplication.java

public static void main(String[] args) throws Exception {
	SpringApplication.run(new Class<?>[0], args);
}

public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
	return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
	// 创建 SpringApplication 对象，并执行运行。
	return new SpringApplication(primarySources).run(args);
}
```

前两个静态方法，最终调用的是第 3 个静态方法。而第 3 个静态方法，实现的逻辑就是：

- 首先，创建一个 SpringApplication 对象。详细的解析，见 [「2.1 构造方法」](http://svip.iocoder.cn/Spring-Boot/SpringApplication/#) 。
- 然后，调用 `SpringApplication#run(Class<?> primarySource, String... args)` 方法，运行 Spring 应用。详细解析，见 [「2.2 run」](http://svip.iocoder.cn/Spring-Boot/SpringApplication/#) 

### 2.1 构造方法

```java
// SpringApplication.java

/**
 * 资源加载器
 */
private ResourceLoader resourceLoader;
/**
 * 主要的 Java Config 类的数组
 */
private Set<Class<?>> primarySources;
/**
 * Web 应用类型
 */
private WebApplicationType webApplicationType;

/**
 * ApplicationContextInitializer 数组
 */
private List<ApplicationContextInitializer<?>> initializers;
/**
 * ApplicationListener 数组
 */
private List<ApplicationListener<?>> listeners;

public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 初始化 initializers 属性
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 初始化 listeners 属性
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

- SpringApplication 的变量比较多，我们先只看构造方法提到的几个。

- `resourceLoader` 属性，资源加载器。可以暂时不理解，感兴趣的胖友，可以看看 [《【死磕 Spring】—— IoC 之 Spring 统一资源加载策略》](http://svip.iocoder.cn/Spring/IoC-load-Resource/?vip) 文章。

- `primarySources` 属性，主要的 Java Config 类的数组。在文初提供的示例，就是 MVCApplication 类。

- `webApplicationType` 属性，调用 `WebApplicationType#deduceFromClasspath()` 方法，通过 classpath ，判断 Web 应用类型。

  - 具体的原理是，是否存在指定的类，艿艿已经在 [WebApplicationType](https://github.com/YunaiV/spring-boot/blob/master/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/WebApplicationType.java) 上的方法添加了注释，直接瞅一眼就明白了。
  - 这个属性，在下面的 `#createApplicationContext()` 方法，将根据它的值（类型），创建不同类型的 ApplicationContext 对象，即 Spring 容器的类型不同。

- `initializers` 属性，ApplicationContextInitializer 数组。

  - 通过 `#getSpringFactoriesInstances(Class<T> type)` 方法，进行获得 ApplicationContextInitializer 类型的对象数组，详细的解析，见 [「2.1.1 getSpringFactoriesInstances」](http://svip.iocoder.cn/Spring-Boot/SpringApplication/#) 方法。
  - 假设只在 Spring MVC 的环境下，`initializers` 属性的结果如下图：

  ![`initializers` 属性](Spring Boot 源码分析 —— SpringApplication.assets/01-1650622691320.jpg)

- `listeners` 属性，ApplicationListener 数组。

  - 也是通过 `#getSpringFactoriesInstances(Class<T> type)` 方法，进行获得 ApplicationListener 类型的对象数组。
  - 假设只在 Spring MVC 的环境下，`listeners` 属性的结果如下图：

  ​	![`listeners` 属性](Spring Boot 源码分析 —— SpringApplication.assets/02-1650622792903.jpg)



`mainApplicationClass` 属性，调用 `#deduceMainApplicationClass()` 方法，获得是调用了哪个 `#main(String[] args)` 方法，代码如下：



```java
// SpringApplication.java

private Class<?> deduceMainApplicationClass() {
	try {
		// 获得当前 StackTraceElement 数组
		StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
		// 判断哪个执行了 main 方法
		for (StackTraceElement stackTraceElement : stackTrace) {
			if ("main".equals(stackTraceElement.getMethodName())) {
				return Class.forName(stackTraceElement.getClassName());
			}
		}
	} catch (ClassNotFoundException ex) {
		// Swallow and continue
	}
	return null;
}
```

- 在文初的例子中，就是 MVCApplication 类。

- 这个 `mainApplicationClass` 属性，没有什么逻辑上的用途，主要就是用来打印下日志，说明是通过这个类启动 Spring 应用的。

#### 2.1.1 getSpringFactoriesInstances

`#getSpringFactoriesInstances(Class<T> type)` 方法，获得指定类类对应的对象们。代码如下：

```java
// SpringApplication.java

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    // <1> 加载指定类型对应的，在 `META-INF/spring.factories` 里的类名的数组
    Set<String> names = new LinkedHashSet<>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // <2> 创建对象们
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    // <3> 排序对象们
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

- `<1>` 处，调用 `SpringFactoriesLoader#loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader)` 方法，加载指定类型对应的，在 `META-INF/spring.factories` 里的类名的数组。

  - 在 [`META-INF/spring.factories`](https://github.com/YunaiV/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/resources/META-INF/spring.factories) 文件中，会以 KEY-VALUE 的格式，配置每个类对应的实现类们。
  - 关于 SpringFactoriesLoader 的该方法，我们就不去细看了。😈 很多时候，我们看源码的时候，不需要陷入到每个方法的细节中。非关键的方法，猜测到具体的用途后，跳过也是没问题的。

- <2>` 处，调用 `#createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args, Set<String> names)` 方法，创建对象们。代码如下：

  ```java
  // SpringApplication.java
  
  /**
   * 创建对象的数组
   *
   * @param type 父类
   * @param parameterTypes 构造方法的参数类型
   * @param classLoader 类加载器
   * @param args 参数
   * @param names 类名的数组
   * @param <T> 泛型
   * @return 对象的数组
   */
  private <T> List<T> createSpringFactoriesInstances(Class<T> type,
  		Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
  		Set<String> names) {
  	List<T> instances = new ArrayList<>(names.size()); // 数组大小，细节~
  	// 遍历 names 数组
  	for (String name : names) {
  		try {
  			// 获得 name 对应的类
  			Class<?> instanceClass = ClassUtils.forName(name, classLoader);
  			// 判断类是否实现自 type 类
  			Assert.isAssignable(type, instanceClass);
  			// 获得构造方法
  			Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
  			// 创建对象
  			T instance = (T) BeanUtils.instantiateClass(constructor, args);
  			instances.add(instance);
  		} catch (Throwable ex) {
  			throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
  		}
  	}
  	return instances;
  }
  ```

- `<3>` 处，调用 `AnnotationAwareOrderComparator#sort(List<?> list)` 方法，排序对象们。例如说，类上有 [`@Order`](https://www.jianshu.com/p/8442d21222ef) 注解。



### 2.2 run

`#run(String... args)` 方法，运行 Spring 应用。代码如下：

```java
// SpringApplication.java

public ConfigurableApplicationContext run(String... args) {
    // <1> 创建 StopWatch 对象，并启动。StopWatch 主要用于简单统计 run 启动过程的时长。
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    //
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // <2> 配置 headless 属性
    configureHeadlessProperty();
    // <3>获得 SpringApplicationRunListener 的数组，并启动监听
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    try {
        // <3> 创建  ApplicationArguments 对象
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // <4> 加载属性配置。执行完成后，所有的 environment 的属性都会加载进来，包括 application.properties 和外部的属性配置。
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);
        // <5> 打印 Spring Banner
        Banner printedBanner = printBanner(environment);
        // <6> 创建 Spring 容器。
        context = createApplicationContext();
        // <7> 异常报告器
        exceptionReporters = getSpringFactoriesInstances(
                SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        // <8> 主要是调用所有初始化类的 initialize 方法
        prepareContext(context, environment, listeners, applicationArguments,
                printedBanner);
        // <9> 初始化 Spring 容器。
        refreshContext(context);
        // <10> 执行 Spring 容器的初始化的后置逻辑。默认实现为空。
        afterRefresh(context, applicationArguments);
        // <11> 停止 StopWatch 统计时长
        stopWatch.stop();
        // <12> 打印 Spring Boot 启动的时长日志。
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // <13> 通知 SpringApplicationRunListener 的数组，Spring 容器启动完成。
        listeners.started(context);
        // <14> 调用 ApplicationRunner 或者 CommandLineRunner 的运行方法。
        callRunners(context, applicationArguments);
    } catch (Throwable ex) {
        // <14.1> 如果发生异常，则进行处理，并抛出 IllegalStateException 异常
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    // <15> 通知 SpringApplicationRunListener 的数组，Spring 容器运行中。
    try {
        listeners.running(context);
    } catch (Throwable ex) {
        // <15.1> 如果发生异常，则进行处理，并抛出 IllegalStateException 异常
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}

```



- `<1>` 处，创建 StopWatch 对象，并调用 `StopWatch#run()` 方法来启动。StopWatch 主要用于简单统计 run 启动过程的时长。

- `<2>` 处，配置 headless 属性。这个逻辑，可以无视，和 AWT 相关。

- `<3>` 处，调用 `#getRunListeners(String[] args)` 方法，获得 SpringApplicationRunListener 数组，并启动监听。代码如下：

  ```java
  // SpringApplication.java
  
  private SpringApplicationRunListeners getRunListeners(String[] args) {
  	Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
  	return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
  			SpringApplicationRunListener.class, types, this, args));
  }
  ```

- 此处的 `listeners` 变量，如下图所示：

  ![`listeners` 属性](Spring Boot 源码分析 —— SpringApplication.assets/03.jpg)

  - 注意噢，此时是 SpringApplication**Run**Listener ，而不是我们看到 `listeners` 的 ApplicationListener 类型。详细的，我们在 [「3. SpringApplicationRunListeners」](http://svip.iocoder.cn/Spring-Boot/SpringApplication/#) 中，在详细解析。

- `<4>` 处，调用 `#prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments)` 方法，加载属性配置。执行完成后，所有的 environment 的属性都会加载进来，包括 `application.properties` 和外部的属性配置。详细的，胖友先一起跳到 [「2.2.1 prepareEnvironment」](http://svip.iocoder.cn/Spring-Boot/SpringApplication/#) 中。

- `<5>` 处，调用 `#printBanner(ConfigurableEnvironment environment)` 方法，打印 Spring Banner 。效果如下：

  ```properties
   .   ____          _            __ _ _
   /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
  ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
   \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
    '  |____| .__|_| |_|_| |_\__, | / / / /
   =========|_|==============|___/=/_/_/_/
   :: Spring Boot ::
  ```

  

- `<6>` 处，调用 `#createApplicationContext()` 方法，创建 Spring 容器。详细解析，见 [「2.2.2 createApplicationContext」](http://svip.iocoder.cn/Spring-Boot/SpringApplication/#) 。

- `<7>` 处，通过 `#getSpringFactoriesInstances(Class<T> type)` 方法，进行获得SpringBootExceptionReporter 类型的对象数组。SpringBootExceptionReporter ，记录启动过程中的异常信息。

  - 此处，`exceptionReporters` 属性的结果如下图：

    ![`listeners` 属性](Spring Boot 源码分析 —— SpringApplication.assets/04.jpg)

- 关于 SpringBootExceptionReporter ，感兴趣的胖友，自己研究先。

- `<8>` 处，调用 `#prepareContext(...)` 方法，主要是调用所有初始化类的 `#initialize(...)` 方法。详细解析，见 [「2.2.3 prepareContext」](http://svip.iocoder.cn/Spring-Boot/SpringApplication/#) 。
- 

