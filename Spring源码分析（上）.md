# Spring源码分析

## 1.注解配置应用上下文类

开始实例化上下文GenericApplicationContext（）的父类中的静态代码块最先被执行

```java
static {
   // Eagerly load the ContextClosedEvent class to avoid weird classloader issues
   // on application shutdown in WebLogic 8.1. (Reported by Dustin Woods.)
   // 在WebLogic 8.1中，急切地加载ContextClosedEvent类，以避免在应用程序关闭时出现奇怪的类加载器问题。		(达斯汀·伍兹报道。).可能他们也不知道啥问题引起的，我理解为没有加载此类，应用程序无法正常关闭
   ContextClosedEvent.class.getName();
}
```

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
   this();
   register(componentClasses);
   refresh();
}
```

### 2.1 this()

```java
public AnnotationConfigApplicationContext() {
   this.reader = new AnnotatedBeanDefinitionReader(this);
   this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```



#### 2.1.1 beanfactory实例化

此时在实例化bean工厂之前调用父类的无参构造方法。

​	

```java
public GenericApplicationContext() {
   this.beanFactory = new DefaultListableBeanFactory();
}
```

在DefaultListableBeanFactory()中调用父类构造方法



```java
public DefaultListableBeanFactory() {
   super();
}
```

连续三个忽略

```java
public AbstractAutowireCapableBeanFactory() {//有自动注入资格的bean工厂的抽象提取类
   super();
   ignoreDependencyInterface(BeanNameAware.class);
   ignoreDependencyInterface(BeanFactoryAware.class);
   ignoreDependencyInterface(BeanClassLoaderAware.class);
}
```

```java
BeanNameAware.class
BeanFactoryAware.class
BeanClassLoaderAware.class
```

```java
/**
 * Ignore the given dependency interface for autowiring.
 * <p>This will typically be used by application contexts to register
 * dependencies that are resolved in other ways, like BeanFactory through
 * BeanFactoryAware or ApplicationContext through ApplicationContextAware.
 * <p>By default, only the BeanFactoryAware interface is ignored.
 * For further types to ignore, invoke this method for each type.
 * @see org.springframework.beans.factory.BeanFactoryAware
 * @see org.springframework.context.ApplicationContextAware
 */
public void ignoreDependencyInterface(Class<?> ifc) {
   this.ignoredDependencyInterfaces.add(ifc);
}
```

忽视这三个接口的特定实现类，如上述的BeanFactoryAware不能通过自动注入获取，会报注入异常，启用由beanfactory callback方式获取相应组件（但是后面的EnvironmentAware我又可以自动注入一个实现类的实例）上面也说的很清楚，默认情况下只有BeanFactoryAware是被忽略的，我们可以理解为开启callback

#### 2.1.2 reader实例化

##### 做了两件重要的事

1.this.registry = registry;

2.conditionEvaluator 用于判断有没有使用@Confitionl注解

3.

```java
AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
//向beanfactory注册内部（internal）各种类
//注册方法如下
AnnotationConfigUtils.registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)
    
    ...
  	private static BeanDefinitionHolder registerPostProcessor(
			BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

		definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(beanName, definition);
		return new BeanDefinitionHolder(definition, beanName);
	}
```

```java
private final AnnotatedBeanDefinitionReader reader;
//Convenient adapter for programmatic registration of bean classes.
//方便的适配器为了bean类的程序化注册

...
this.reader = new AnnotatedBeanDefinitionReader(this);
```

  **BeanDefinitionRegistry是AnnotationConfigApplicationContext的父类	GenericApplicationContext实现的接口，（注册中心）用来保存bean定义接口（例如BeanDefinition 实现类RootBeanDefinition和ChildBeanDefinition实例）的，通常实现BeanDefinitionRegistry的beanfactory实现（GenericApplicationContext实现了BeanDefinitionRegistry，它的实例属性beanfactory（DefaultListableBeanFactory类型）也实现了BeanDefinitionRegistry，beanfactory底层使用默认256个容量的ConcurrentHashMap beanDefinitionMap实现bean存储）。BeanDefinition ：bean定义描述了一个bean实例，该实例具有属性值、构造函数参数值和由具体实现提供的进一步信息。**

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
   this(registry, getOrCreateEnvironment(registry));
}
```



getOrCreateEnvironment方法创建标准环境

```java
private static Environment getOrCreateEnvironment(BeanDefinitionRegistry registry) {
   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   if (registry instanceof EnvironmentCapable) {
      return ((EnvironmentCapable) registry).getEnvironment();
   }
   return new StandardEnvironment();
}
```

接下来就算是reader（带注解的bean定义读取器）实例初始化的过程

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   Assert.notNull(environment, "Environment must not be null");
   this.registry = registry;
    //Internal class used to evaluate {@link Conditional} annotations.
    //conditionEvaluator用于判断有没有使用@Confitionl注解
   this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
   AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

此类构造方法里先根据org.springframework.util包下的Assert（断言）类判断registry和环境是否为空

并将AnnotationConfigApplicationContext.reader设置为AnnotatedBeanDefinitionReader，AnnotatedBeanDefinitionReader.registry = AnnotationConfigApplicationContext可以看成相互依赖引用



最后一步登记 注解配置处理器

```java
public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
   registerAnnotationConfigProcessors(registry, null);
}
```



```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
      BeanDefinitionRegistry registry, @Nullable Object source) {
	//获取并强转registry为beanfactory的类型，unwrapDefaultListableBeanFactory里面判断是哪个类型DefaultListableBeanFactory，GenericApplicationContext，都不是返回空
   DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
   if (beanFactory != null) {
      if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
         beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
      }
      if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
         beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
      }
   }

   Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

   if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
     
      RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
      def.setSource(source);
        //bean名称 "org.springframework.context.annotation.internalConfigurationAnnotationProcessor";
      beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
   if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
   if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition();
      try {
         def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
               AnnotationConfigUtils.class.getClassLoader()));
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
      }
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
   }

   return beanDefs;
}
```

上面方法用于登记注解配置处理器：

```java
public static final String CONFIGURATION_BEAN_NAME_GENERATOR =
      "org.springframework.context.annotation.internalConfigurationBeanNameGenerator";

/**
 * The bean name of the internally managed Autowired annotation processor.
 */
public static final String AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME =
      "org.springframework.context.annotation.internalAutowiredAnnotationProcessor";

/**
 * The bean name of the internally managed Required annotation processor.
 * @deprecated as of 5.1, since no Required processor is registered by default anymore
 */
@Deprecated
public static final String REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME =
      "org.springframework.context.annotation.internalRequiredAnnotationProcessor";

/**
 * The bean name of the internally managed JSR-250 annotation processor.
 */
public static final String COMMON_ANNOTATION_PROCESSOR_BEAN_NAME =
      "org.springframework.context.annotation.internalCommonAnnotationProcessor";

/**
 * The bean name of the internally managed JPA annotation processor.
 */
public static final String PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME =
      "org.springframework.context.annotation.internalPersistenceAnnotationProcessor";

private static final String PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME =
      "org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor";

/**
 * The bean name of the internally managed @EventListener annotation processor.
 */
public static final String EVENT_LISTENER_PROCESSOR_BEAN_NAME =
      "org.springframework.context.event.internalEventListenerProcessor";

/**
 * The bean name of the internally managed EventListenerFactory.
 */
public static final String EVENT_LISTENER_FACTORY_BEAN_NAME =
      "org.springframework.context.event.internalEventListenerFactory";
```

#### 2.1.3 scanner的实例化（尚未探究）

```java
this.scanner = new ClassPathBeanDefinitionScanner(this);
```

主要作用：

1.设置默认包扫描过滤器

2.设置默认带有过滤器的资源加载器，防止后面@configration（可能不止主配置类）没有配置过滤器。

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
   this(registry, true);//第二个参数defaultfilter，默认过滤器扫描所有的注解
}

```

```
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
   this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
}
```

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
		Environment environment) {

	this(registry, useDefaultFilters, environment,
			(registry instanceof ResourceLoader ? (ResourceLoader) registry : null));
}
```
```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
      Environment environment, @Nullable ResourceLoader resourceLoader) {

   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   this.registry = registry;

   if (useDefaultFilters) {
      registerDefaultFilters();//默认过滤器  所有注册bean的注解都扫描
   }
   setEnvironment(environment);//设置环境
   setResourceLoader(resourceLoader);//设置默认带有过滤器的资源加载器，和扫描类路径下*.class的文件
    //防止后面@configration没有配置过滤器。
}
```

## 2.2 register(componentClasses);

componentClasses是一个class数组也就是@Configuration注解的类一般传入主注解类文件

```java
this.reader.register(componentClasses);
```

```java
public void register(Class<?>... componentClasses) {
   for (Class<?> componentClass : componentClasses) {
      registerBean(componentClass);
   }
}
```

```java
public void registerBean(Class<?> beanClass) {
   doRegisterBean(beanClass, null, null, null);
}
```

```java
<T> void doRegisterBean(Class<T> beanClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
      @Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {
	//AnnotatedGenericBeanDefinition里面封装了配置类的信息如注解信息等
   AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
   if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
       //This()里实例化的conditionEvaluator判断abd里面有没有 Conditional注解有就继续处理Conditional接口的拦截方法，没有就返回 flase，目前没用到，以后可继续往下探究，一步步分析并不难
       //此if跳过
      return;
   }

   abd.setInstanceSupplier(instanceSupplier);
   ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
   abd.setScope(scopeMetadata.getScopeName());
    //生成默认的bean名字首字母小写
   String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
	//里面判断adb有没有如下注解,有就获取值并设置进 AnnotatedGenericBeanDefinition的实例属性
    //Lazy.class, Primary.class,DependsOn.class,Role.class, Description.class
   AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
   if (qualifiers != null) {
      for (Class<? extends Annotation> qualifier : qualifiers) {
         if (Primary.class == qualifier) {
            abd.setPrimary(true);
         }
         else if (Lazy.class == qualifier) {
            abd.setLazyInit(true);
         }
         else {
            abd.addQualifier(new AutowireCandidateQualifier(qualifier));
         }
      }
   }
    //BeanDefinitionCustomizer是一个函数式接口
   for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
      customizer.customize(abd);
   }
//注册bean BeanDefinitionReaderUtils这个工具类里面的方法很重要
   BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
   definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
   BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

