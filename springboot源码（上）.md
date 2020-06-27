# springboot源码

**先记录一下重要的几个监听器：**

- ApplicationContextInitializer.class
- SpringApplicationRunListener.class
- ApplicationRunner.class
- CommandLineRunner.class



```java
SpringApplication.run(StudySpringbootApplication.class, args);
```

## 1.new SpringApplication（）

```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
   return run(new Class<?>[] { primarySource }, args);
}
```

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
   return new SpringApplication(primarySources).run(args);
}
```

```java
public SpringApplication(Class<?>... primarySources) {
   this(null, primarySources);
}
```

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
   this.resourceLoader = resourceLoader;
   Assert.notNull(primarySources, "PrimarySources must not be null");
    //1.1保存在SpringApplication入口类的主要资源类是一个set集合，你可以传入多个，后面会根据你传的类进行main方法的分析
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //1.2从类路径下推断是不是web应用及web应用的类型
   this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //1.3 关于ApplicationContextInitializer.class接口的部分注释：Callback interface for initializing a Spring {@link ConfigurableApplicationContext} prior to being {@linkplain ConfigurableApplicationContext#refresh() refreshed}.大致意思是，在ConfigurableApplicationContext刷新前初始化ConfigurableApplicationContext的回调接口，很重要，很重要，很重要...
   setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    //1.4 找到ApplicationListener.class的实现类，方法同上
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
 	//1.5 确定 main方法在你传进来的那个类中
   this.mainApplicationClass = deduceMainApplicationClass();
}
```

进入上面**1.2** deduceFromClasspath（）方法：

```java
static WebApplicationType deduceFromClasspath() {
    //1.2.1判断"org.springframework.web.reactive.DispatcherHandler"，"org.springframework.web.servlet.DispatcherServlet"，"org.glassfish.jersey.servlet.ServletContainer"是不是存在，都存在返回一个web类型REACTIVE（具体可以去官网了解WebFlux）简单介绍一下spring mvc是基于Servlet Api的，所以是同步阻塞I/O模型，每一个请求都要一个线程去处理，Spring WebFlux 是一个异步非阻塞式的 Web 框架，它能够充分利用多核 CPU 的硬件资源去处理大量的并发请求。所以，它特别适合应用在 IO 密集型的服务中，比如微服务网关这样的应用中。
    //当然，作为一家人都可以使用自家注解，springmvc调试更简单，不到万不得已，推荐spring mvc
    
    //判断方法：Class.forName(name, false, clToUse)，有类就没异常，返回true，没有就进异常处理catch，然后继续执行（比你直接写在类路径下通过io查找方便许多，不知道速度怎么样，有时间测试下）
   if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
         && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
      return WebApplicationType.REACTIVE;
   }
    //1.2.2 	private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet","org.springframework.web.context.ConfigurableWebApplicationContext" };判断这两个类是不是在类路径下，没导web-starter包就没有，有就返回servlet(大家最熟悉的)
   for (String className : SERVLET_INDICATOR_CLASSES) {
      if (!ClassUtils.isPresent(className, null)) {
         return WebApplicationType.NONE;
      }
   }
   return WebApplicationType.SERVLET;
}
```

进入1.3 getSpringFactoriesInstances(ApplicationContextInitializer.class)方法：

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
   return getSpringFactoriesInstances(type, new Class<?>[] {});
}
```

进入

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
   ClassLoader classLoader = getClassLoader();
   // Use names and ensure unique to protect against duplicates
    //使用名字并确保唯一，防止重复
    //1.3.1 
   Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    //1.3.2 实例化上面得到的ApplicationContextInitializer.class的实现类
   List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    //排个序，，可能是默认ApplicationContextInitializer.class之间有依赖关系吧
   AnnotationAwareOrderComparator.sort(instances);
   return instances;
}
```

进入**1.3.1** SpringFactoriesLoader.loadFactoryNames(type, classLoader)方法：

```java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
   String factoryTypeName = factoryType.getName();
   return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}
```

进入loadSpringFactories(classLoader)方法：

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
   MultiValueMap<String, String> result = cache.get(classLoader);
   if (result != null) {
      return result;
   }

   try {
      
	/**
	 * The location to look for factories.在本地jar包的META-INF/spring.factories中寻找工厂类，可能存在多个jar包中。
	 * <p>Can be present in multiple JAR files.
	 */
//	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
                               //classloader会默认从classpath中查找该文件。返回一个CompoundEnumeration<>(tmp）
       Enumeration<URL> urls = (classLoader != null ?
            classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
            ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
      result = new LinkedMultiValueMap<>();
       //将上面得到的CompoundEnumeration<>(tmp）中缓存的东西放到LinkedMultiValueMap<>()中
      while (urls.hasMoreElements()) {
         URL url = urls.nextElement();
         UrlResource resource = new UrlResource(url);
         Properties properties = PropertiesLoaderUtils.loadProperties(resource);
         for (Map.Entry<?, ?> entry : properties.entrySet()) {
            String factoryTypeName = ((String) entry.getKey()).trim();
            for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
               result.add(factoryTypeName, factoryImplementationName.trim());
            }
         }
      }
      cache.put(classLoader, result);
      return result;
   }
   catch (IOException ex) {
      throw new IllegalArgumentException("Unable to load factories from location [" +
            FACTORIES_RESOURCE_LOCATION + "]", ex);
   }
}
```

进入**1.3** 的setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class))方法：

```java
public void setInitializers(Collection<? extends ApplicationContextInitializer<?>> initializers) {
    //可以看到这些重要的initializers存在SpringApplication的initializers 的List集合中。
   this.initializers = new ArrayList<>(initializers);
}
```

进入 **1.5 **的deduceMainApplicationClass()方法：

```java
private Class<?> deduceMainApplicationClass() {
   try {
       //StackTraceElement 数组里面保存的是当前线程的堆栈方法调用信息
      StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
      for (StackTraceElement stackTraceElement : stackTrace) {
         if ("main".equals(stackTraceElement.getMethodName())) {
            return Class.forName(stackTraceElement.getClassName());
         }
      }
   }
   catch (ClassNotFoundException ex) {
      // Swallow and continue
   }
   return null;
}
```

**到此SpringApplication这个入口类就被实例化完成了，总结一下**：

- 1.1保存在SpringApplication入口类的主要资源类是一个set集合，你可以传入多个，后面会根据你传的类进行main方法的分析
- 1.2从类路径下推断是不是web应用及web应用的类型
- 1.3 从META-INF/spring.factories里找到ApplicationContextInitializer.class接口的实现类
- 1.4 找到ApplicationListener.class的实现类，方法同上
- 1.5确定 main方法在你传进来的主要资源类的哪个类中