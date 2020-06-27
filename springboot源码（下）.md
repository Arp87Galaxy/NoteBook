# springboot源码（下）

**重要的注解处理器：**

在bean定义时期将组合注解分解存入matedata

```java
TypeMappedAnnotations //spliterator(null)里调用getAggregates（）里面再调用scan方法获取组合的注解
```

```java
invokeBeanFactoryPostProcessors(beanFactory);//这一步会解析配置类
```

第一步先扫描包：

```java
 ConfigurationClassPostProcessor//在这个类的processConfigBeanDefinitions方法中
```

会定义一个

```java
ConfigurationClassParser parser;
```

```java
parser.parse(candidates);//调用这个方法，进入
```

```java
parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());//再调用这个方法
```

里面先包扫描得到你配置的非@bean类然后再扫描所有类看看是不是用import（组合注解在bean定义时候就解析完成）引入了类

```java
 processImports//在这个方法中处理得到所有
```

```java
this.deferredImportSelectorHandler.process();//延迟调用selector（注解EnableAutoConfigtion注解里面的类）的getAutoConfigurationEntry方法
```

导入所有自动配置类

完成











**先记录一下重要的几个监听器：**

- ApplicationContextInitializer.class
- SpringApplicationRunListener.class
- ApplicationRunner.class
- CommandLineRunner.class

```java
public ConfigurableApplicationContext run(String... args) {
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
    //spring里面的入口类，spring容器
   ConfigurableApplicationContext context = null;
    //异常记录器
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    //jwt相关的，用的时候再看吧
   configureHeadlessProperty();
    //2.1 里面的具体获取方法同上一篇一样new SpringApplicationRunListeners(logger,getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
    //看到getSpringFactoriesInstances（）方法就是从META-INF/spring.factories下获取到SpringApplicationRunListener.class
   SpringApplicationRunListeners listeners = getRunListeners(args);
    //回调SpringApplicationRunListener的starting方法，默认的EventPublishingRunListener已经包含一个事件多播器也就是派发器,不像spring事先存在集合里，最后实例化其余bean之前才开始播。springboot在bean定义前就定义好派发器了。
   listeners.starting();
    
   try {
       //封装了一下启动时的命令行参数
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
       //准备环境
      ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
      configureIgnoreBeanInfo(environment);
      Banner printedBanner = printBanner(environment);
      context = createApplicationContext();
      exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
      prepareContext(context, environment, listeners, applicationArguments, printedBanner);
      refreshContext(context);
      afterRefresh(context, applicationArguments);
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
      }
      listeners.started(context);
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```