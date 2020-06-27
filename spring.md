



# 配置注解

### 1.@ComponentScans可以指定过滤规则，没有指定使用默认的会扫描所有

```java
	package com.atguigu.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.FilterType;
import org.springframework.context.annotation.ComponentScan.Filter;
import org.springframework.context.annotation.ComponentScans;

import com.atguigu.bean.Person;

//配置类==配置文件
@Configuration  //告诉Spring这是一个配置类

@ComponentScans(
		value = {
				@ComponentScan(value="com.atguigu",includeFilters = {
/*						@Filter(type=FilterType.ANNOTATION,classes={Controller.class}),
						@Filter(type=FilterType.ASSIGNABLE_TYPE,classes={BookService.class}),*/
						@Filter(type=FilterType.CUSTOM,classes={MyTypeFilter.class})
				},useDefaultFilters = false)	
		}
		)
//@ComponentScan  value:指定要扫描的包
//excludeFilters = Filter[] ：指定扫描的时候按照什么规则排除那些组件
//includeFilters = Filter[] ：指定扫描的时候只需要包含哪些组件,因为默认的过滤是包含所有，而include在默认里面所以useDefaultFilters=false才能生效
//FilterType.ANNOTATION：按照注解
//FilterType.ASSIGNABLE_TYPE：按照给定的类型；
//FilterType.ASPECTJ：使用ASPECTJ表达式
//FilterType.REGEX：使用正则指定
//FilterType.CUSTOM：使用自定义规则
public class MainConfig {
	
	//给容器中注册一个Bean;类型为返回值的类型，id默认是用方法名作为id
	@Bean("person")
	public Person person01(){
		return new Person("lisi", 20);
	}

}
```



### 2.注册bean配置注解@Scope，@Lazy ，@Bean，导入逐渐注解@Import({Color.class,Red.class,MyImportSelector.class,MyImportBeanDefinitionRegistrar.class})，条件注解 满足条件才生效@Conditional({WindowsCondition.class})，@ConditionalOnMiss等利用FactoryBean注册bean

```java
package com.atguigu.config;

import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Conditional;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.Lazy;
import org.springframework.context.annotation.Scope;

import com.atguigu.bean.Blue;
import com.atguigu.bean.Color;
import com.atguigu.bean.ColorFactoryBean;
import com.atguigu.bean.Person;
import com.atguigu.bean.Red;
import com.atguigu.condition.LinuxCondition;
import com.atguigu.condition.MyImportBeanDefinitionRegistrar;
import com.atguigu.condition.MyImportSelector;
import com.atguigu.condition.WindowsCondition;
import com.atguigu.test.IOCTest;

//类中组件统一设置。满足当前条件，这个类中配置的所有bean注册才能生效；
@Conditional({WindowsCondition.class})
@Configuration
@Import({Color.class,Red.class,MyImportSelector.class,MyImportBeanDefinitionRegistrar.class})
//@Import导入组件，id默认是组件的全类名
public class MainConfig2 {
   
   //默认是单实例的
   /**
    * ConfigurableBeanFactory#SCOPE_PROTOTYPE    
    * @see ConfigurableBeanFactory#SCOPE_SINGLETON  
    * @see org.springframework.web.context.WebApplicationContext#SCOPE_REQUEST  request
    * @see org.springframework.web.context.WebApplicationContext#SCOPE_SESSION     sesssion
    * @return\
    * @Scope:调整作用域
    * prototype：多实例的：ioc容器启动并不会去调用方法创建对象放在容器中。
    *                 每次获取的时候才会调用方法创建对象；
    * singleton：单实例的（默认值）：ioc容器启动会调用方法创建对象放到ioc容器中。
    *           以后每次获取就是直接从容器（map.get()）中拿，
    * request：同一次请求创建一个实例
    * session：同一个session创建一个实例
    * 
    * 懒加载：
    *        单实例bean：默认在容器启动的时候创建对象；
    *        懒加载：容器启动不创建对象。第一次使用(获取)Bean创建对象，并初始化；
    * 
    */
// @Scope("prototype")
   @Lazy
   @Bean("person")
   public Person person(){
      System.out.println("给容器中添加Person....");
      return new Person("张三", 25);
   }
   
   /**
    * @Conditional({Condition}) ： 按照一定的条件进行判断，满足条件给容器中注册bean
    * 
    * 如果系统是windows，给容器中注册("bill")
    * 如果是linux系统，给容器中注册("linus")
    */
   
   @Bean("bill")
   public Person person01(){
      return new Person("Bill Gates",62);
   }
   
   @Conditional(LinuxCondition.class)
   @Bean("linus")
   public Person person02(){
      return new Person("linus", 48);
   }
   
   /**
    * 给容器中注册组件；
    * 1）、包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]
    * 2）、@Bean[导入的第三方包里面的组件]
    * 3）、@Import[快速给容器中导入一个组件]
    *        1）、@Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
    *        2）、ImportSelector:返回需要导入的组件的全类名数组；
    *        3）、ImportBeanDefinitionRegistrar:手动注册bean到容器中
    * 4）、使用Spring提供的 FactoryBean（工厂Bean）;
    *        1）、默认获取到的是工厂bean调用getObject创建的对象
    *        2）、要获取工厂Bean本身，我们需要给id前面加一个&
    *           &colorFactoryBean
    */
   @Bean
   public ColorFactoryBean colorFactoryBean(){
      return new ColorFactoryBean();
   }
}
```

####  条件类

```java
package com.atguigu.condition;

import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotatedTypeMetadata;

//判断是否windows系统
public class WindowsCondition implements Condition {

   @Override
   public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
      Environment environment = context.getEnvironment();
      String property = environment.getProperty("os.name");
      if(property.contains("Windows")){
         return true;
      }
      return false;
   }

}

package com.atguigu.condition;

import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotatedTypeMetadata;

//判断是否linux系统
public class LinuxCondition implements Condition {

	/**
	 * ConditionContext：判断条件能使用的上下文（环境）
	 * AnnotatedTypeMetadata：注释信息
	 */
	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		// TODO是否linux系统
		//1、能获取到ioc使用的beanfactory
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		//2、获取类加载器
		ClassLoader classLoader = context.getClassLoader();
		//3、获取当前环境信息
		Environment environment = context.getEnvironment();
		//4、获取到bean定义的注册类
		BeanDefinitionRegistry registry = context.getRegistry();
		
		String property = environment.getProperty("os.name");
		
		//可以判断容器中的bean注册情况，也可以给容器中注册bean
		boolean definition = registry.containsBeanDefinition("person");
		if(property.contains("linux")){
			return true;
		}
		
		return false;
	}

}

```

#### 工厂bean注册bean

```java
package com.atguigu.bean;

import org.springframework.beans.factory.FactoryBean;

//创建一个Spring定义的FactoryBean
public class ColorFactoryBean implements FactoryBean<Color> {

   //返回一个Color对象，这个对象会添加到容器中
   @Override
   public Color getObject() throws Exception {
      // TODO Auto-generated method stub
      System.out.println("ColorFactoryBean...getObject...");
      return new Color();
   }

   @Override
   public Class<?> getObjectType() {
      // TODO Auto-generated method stub
      return Color.class;
   }

   //是单例？
   //true：这个bean是单实例，在容器中保存一份
   //false：多实例，每次获取都会创建一个新的bean；
   @Override
   public boolean isSingleton() {
      // TODO Auto-generated method stub
      return false;
   }

}
```

### bean生命周期

```java
package com.atguigu.config;

import org.springframework.context.ApplicationListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;

import com.atguigu.bean.Car;

/**
 * bean的生命周期：
 *        bean创建---实例化(new)BeanWrapper中---赋值(这一步还会自动注入依赖)---初始化(赋值操作，如数据源的连接信息等)---销毁的过程
 * 容器管理bean的生命周期；
 * 我们可以自定义初始化和销毁方法；容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法
 * 
 * 构造（对象创建）
 *        单实例：在容器启动的时候创建对象
 *        多实例：在每次获取的时候创建对象\
 * 
 * BeanPostProcessor.postProcessBeforeInitialization
 * 初始化：
 *        对象创建完成，并赋值好，调用初始化方法。。。
 * BeanPostProcessor.postProcessAfterInitialization
 * 销毁：
 *        单实例：容器关闭的时候
 *        多实例：容器不会管理这个bean；容器不会调用销毁方法；
 * 
 * 
 * 遍历得到容器中所有的BeanPostProcessor；挨个执行beforeInitialization，
 * 一但返回null，跳出for循环，不会执行后面的BeanPostProcessor.postProcessorsBeforeInitialization
 * 
 * BeanPostProcessor原理
 * populateBean(beanName, mbd, instanceWrapper);给bean进行属性赋值
 * initializeBean
 * {
 * applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
 * invokeInitMethods(beanName, wrappedBean, mbd);执行自定义初始化
 * applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
 *}
 * 
 * 
 * 
 * 1）、指定初始化和销毁方法；
 *        通过@Bean指定init-method和destroy-method；
 * 2）、通过让Bean实现InitializingBean（定义初始化逻辑）接口，
 *              DisposableBean（定义销毁逻辑，关闭连接等）接口;
 * 3）、可以使用JSR250；
 *        @PostConstruct：在bean创建完成并且属性赋值完成后；来执行初始化方法
 *        @PreDestroy：在容器销毁bean之前通知我们进行清理工作
 * 4）、BeanPostProcessor【interface】：bean的后置处理器；
 *        在bean初始化前后进行一些处理工作；
 *        postProcessBeforeInitialization:在初始化之前工作
 *        postProcessAfterInitialization:在初始化之后工作
 * 
 * Spring底层对 BeanPostProcessor 的使用；
 *        bean赋值，注入其他组件，@Autowired，生命周期注解功能，@Async,xxx BeanPostProcessor;
 * 
 * @author lfy
 *
 */
@ComponentScan("com.atguigu.bean")
@Configuration
public class MainConfigOfLifeCycle {
   
   //@Scope("prototype")
   @Bean(initMethod="init",destroyMethod="detory")
   public Car car(){
      return new Car();
   }

}
```

### 配置文件取值，引入配置文件



```java
//使用@Value赋值；
//1、基本数值
//2、可以写SpEL； #{}计算值
//3、可以写${}；取出配置文件【properties】中的值（在运行环境变量里面的值）

@Value("张三")
private String name;
@Value("#{20-2}")
private Integer age;

@Value("${person.nickName}")
private String nickName;
@Value("${person.nickName}:${person.nick:zs}")//: 表示没有前面的用后面的代替
private String nickName;




@ConfigurationProperties(prifix="person")自动配置通过validate的外部配置
@ConfigurationPropertiesScan（整个包的自动扫描并配置）
@ConstructorBinding（调用setter方法来执行配置）
@ConfigurationPropertiesBindingPostProcessor没看
@EnableConfigurationProperties没看
```

### 开发环境，生产环境

```java
package com.atguigu.config;


import javax.sql.DataSource;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.EmbeddedValueResolverAware;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.context.annotation.PropertySource;
import org.springframework.util.StringValueResolver;

import com.atguigu.bean.Yellow;
import com.mchange.v2.c3p0.ComboPooledDataSource;

/**
 * Profile：
 *        Spring为我们提供的可以根据当前环境，动态的激活和切换一系列组件的功能；
 * 
 * 开发环境、测试环境、生产环境；
 * 数据源：(/A)(/B)(/C)；
 * 
 * 
 * @Profile：指定组件在哪个环境的情况下才能被注册到容器中，不指定，任何环境下都能注册这个组件
 * 
 * 1）、加了环境标识的bean，只有这个环境被激活的时候才能注册到容器中。默认是default环境
 * 2）、写在配置类上，只有是指定的环境的时候，整个配置类里面的所有配置才能开始生效
 * 3）、没有标注环境标识的bean在，任何环境下都是加载的；
 */

@PropertySource("classpath:/dbconfig.properties")
@Configuration
public class MainConfigOfProfile implements EmbeddedValueResolverAware{
   
   @Value("${db.user}")
   private String user;
   
   private StringValueResolver valueResolver;
   
   private String  driverClass;
   
   
   @Bean
   public Yellow yellow(){
      return new Yellow();
   }
   
   @Profile("test")
   @Bean("testDataSource")
   public DataSource dataSourceTest(@Value("${db.password}")String pwd) throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser(user);
      dataSource.setPassword(pwd);
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/test");
      dataSource.setDriverClass(driverClass);
      return dataSource;
   }
   
   
   @Profile("dev")
   @Bean("devDataSource")
   public DataSource dataSourceDev(@Value("${db.password}")String pwd) throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser(user);
      dataSource.setPassword(pwd);
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/ssm_crud");
      dataSource.setDriverClass(driverClass);
      return dataSource;
   }
   
   @Profile("prod")
   @Bean("prodDataSource")
   public DataSource dataSourceProd(@Value("${db.password}")String pwd) throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser(user);
      dataSource.setPassword(pwd);
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/scw_0515");
      
      dataSource.setDriverClass(driverClass);
      return dataSource;
   }

   @Override
   public void setEmbeddedValueResolver(StringValueResolver resolver) {
      // TODO Auto-generated method stub
      this.valueResolver = resolver;
      driverClass = valueResolver.resolveStringValue("${db.driverClass}");
   }

}
```

### 自动装配@Autowired (对IOC容器属性DI)  @Qualifier("bean名")      @Primary在没有Qualifier时装配时默认的bean，@Resource，@Inject

```
package com.atguigu.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import com.atguigu.bean.Car;
import com.atguigu.bean.Color;
import com.atguigu.dao.BookDao;


/**
 * 自动装配;
 *        Spring利用依赖注入（DI），完成对IOC容器中中各个组件的依赖关系赋值；
 * 
 * 1）、@Autowired：自动注入：
 *        1）、默认优先按照类型去容器中找对应的组件:applicationContext.getBean(BookDao.class);找到就赋值
 *        2）、如果找到多个相同类型的组件，再将属性的名称作为组件的id去容器中查找
 *                       applicationContext.getBean("bookDao")
 *        3）、@Qualifier("bookDao")：使用@Qualifier指定需要装配的组件的id，而不是使用属性名
 *        4）、自动装配默认一定要将属性赋值好，没有就会报错；
 *           可以使用@Autowired(required=false);
 *        5）、@Primary：让Spring进行自动装配的时候，默认使用首选的bean；
 *              也可以继续使用@Qualifier指定需要装配的bean的名字
 *        BookService{
 *           @Autowired
 *           BookDao  bookDao;
 *        }
 * 
 * 2）、Spring还支持使用@Resource(JSR250)和@Inject(JSR330)[java规范的注解]
 *        @Resource:
 *           可以和@Autowired一样实现自动装配功能；默认是按照组件名称进行装配的；
 *           没有能支持@Primary功能没有支持@Autowired（reqiured=false）;
 *        @Inject:
 *           需要导入javax.inject的包，和Autowired的功能一样。没有required=false的功能；
 *  @Autowired:Spring定义的； @Resource、@Inject都是java规范
 *     
 * AutowiredAnnotationBeanPostProcessor:解析完成自动装配功能；       
 * 
 * 3）、 @Autowired:构造器，参数，方法，属性；都是从容器中获取参数组件的值
 *        1）、[标注在方法位置]：@Bean+方法参数；参数从容器中获取;默认不写@Autowired效果是一样的；都能自动装配
 *        2）、[标在构造器上]：如果组件只有一个有参构造器，这个有参构造器的@Autowired可以省略，参数位置的组件还是可以自动从容器中获取
 *        3）、放在参数位置：
 * 
 * 4）、自定义组件想要使用Spring容器底层的一些组件（ApplicationContext，BeanFactory，xxx）；
 *        自定义组件实现xxxAware；在创建对象的时候，会调用接口规定的方法注入相关组件；Aware；
 *        把Spring底层一些组件注入到自定义的Bean中；
 *        xxxAware：功能使用xxxProcessor；
 *           ApplicationContextAware==》ApplicationContextAwareProcessor；
 *     
 *        
 * @author lfy
 *
 */
@Configuration
@ComponentScan({"com.atguigu.service","com.atguigu.dao",
   "com.atguigu.controller","com.atguigu.bean"})
public class MainConifgOfAutowired {
   
   @Primary
   @Bean("bookDao2")
   public BookDao bookDao(){
      BookDao bookDao = new BookDao();
      bookDao.setLable("2");
      return bookDao;
   }
   
   /**
    * @Bean标注的方法创建对象的时候，方法参数的值从容器中获取
    * @param car
    * @return
    */
   @Bean
   public Color color(Car car){
      Color color = new Color();
      color.setCar(car);
      return color;
   }
   

}
```



# Aware

```java
package com.atguigu.bean;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.EmbeddedValueResolverAware;
import org.springframework.stereotype.Component;
import org.springframework.util.StringValueResolver;

@Component
public class Red implements ApplicationContextAware,BeanNameAware,EmbeddedValueResolverAware {
   
   private ApplicationContext applicationContext;

   @Override
   public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
      // TODO Auto-generated method stub
      System.out.println("传入的ioc："+applicationContext);
      this.applicationContext = applicationContext;
   }

   @Override
   public void setBeanName(String name) {
      // TODO Auto-generated method stub
      System.out.println("当前bean的名字："+name);
   }

   @Override
   public void setEmbeddedValueResolver(StringValueResolver resolver) {
      // TODO Auto-generated method stub
      String resolveStringValue = resolver.resolveStringValue("你好 ${os.name} 我是 #{20*18}");
      System.out.println("解析的字符串："+resolveStringValue);
   }




}
```

# IOC容器



**This chapter covers Spring’s Inversion of Control (IoC) container.**

**这章包含spring的控制反转容器**

### 1.1 Spring IoC容器和Bean简介

IoC时一种java设计原则，实现方式有依赖注入（DI）和依赖查找DL。

通过控制反转，对象在被创建的时候，由一个调控系统内所有对象的外界实体将其所依赖的对象的引用传递给它。对象仅通过构造函数参数，工厂方法的参数或在构造或从工厂方法返回后在对象实例上设置的属性来定义其依赖项（即，与它们一起使用的其他对象） 。然后，容器在创建bean时注入那些依赖项。此过程从根本上讲是通过使用类的直接构造或诸如服务定位器模式之类的方法来控制其依赖项的实例化或位置的bean本身的逆过程（因此称为Control Inversion）。

简单点说就是，以前我们都是在构造方法参数中传入，或者在setter方法参数中传入已经创建好的实例，现在我们只用设置需要的对象的实例属性，其他则交给ioc容器管理。

#### 1.1.1什么是IOC

​	从spring源码底层可以看出，在对已经定义好的bean定义实例化时，我们在注入属性时，会去查找依赖的bean定义是否已经实例化，没有则先实例化依赖的实例。不像以前，我们都时先实例化好依赖的实例，然后再以参数的形式传给当前bean去创建当前bean实例

### 1.2 bean



# AOP原理

### 1.题目

1. spring aop 与 aspectj的区别

   spring aop 运行期间通过动态代理方式，aspectj是编译期间 ，通过织入的方式。

   由于动态代理 会间接的调用 被代理的方法，因此不适用于非public方法。可以考虑aspectj

```java
package com.atguigu.config;



import org.aopalliance.aop.Advice;
import org.aopalliance.intercept.MethodInterceptor;
import org.springframework.aop.Advisor;
import org.springframework.aop.Pointcut;
import org.springframework.aop.framework.AopInfrastructureBean;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

import com.atguigu.aop.LogAspects;
import com.atguigu.aop.MathCalculator;

/**
 * AOP：【动态代理】
 *        指在程序运行期间动态的将某段代码切入到指定方法指定位置进行运行的编程方式；
 * 
 * 1、导入aop模块；Spring AOP：(spring-aspects)
 * 2、定义一个业务逻辑类（MathCalculator）；在业务逻辑运行的时候将日志进行打印（方法之前、方法运行结束、方法出现异常，xxx）
 * 3、定义一个日志切面类（LogAspects）：切面类里面的方法需要动态感知MathCalculator.div运行到哪里然后执行；
 *        通知方法：
 *           前置通知(@Before)：logStart：在目标方法(div)运行之前运行
 *           后置通知(@After)：logEnd：在目标方法(div)运行结束之后运行（无论方法正常结束还是异常结束）
 *           返回通知(@AfterReturning)：logReturn：在目标方法(div)正常返回之后运行
 *           异常通知(@AfterThrowing)：logException：在目标方法(div)出现异常以后运行
 *           环绕通知(@Around)：动态代理，手动推进目标方法运行（joinPoint.procced()）
 * 4、给切面类的目标方法标注何时何地运行（通知注解）；
 * 5、将切面类和业务逻辑类（目标方法所在类）都加入到容器中;
 * 6、必须告诉Spring哪个类是切面类(给切面类上加一个注解：@Aspect)
 * [7]、给配置类中加 @EnableAspectJAutoProxy 【开启基于注解的aop模式】
 *        在Spring中很多的 @EnableXXX;
 * 
 * 三步：
 *     1）、将业务逻辑组件和切面类都加入到容器中；告诉Spring哪个是切面类（@Aspect）
 *     2）、在切面类上的每一个通知方法上标注通知注解，告诉Spring何时何地运行（切入点表达式）
 *  3）、开启基于注解的aop模式；@EnableAspectJAutoProxy
 *  
 * AOP原理：【看给容器中注册了什么组件，这个组件什么时候工作，这个组件的功能是什么？】
 *        @EnableAspectJAutoProxy；
 * 1、@EnableAspectJAutoProxy是什么？
 *        @Import(AspectJAutoProxyRegistrar.class)：给容器中导入AspectJAutoProxyRegistrar
 *           利用AspectJAutoProxyRegistrar自定义给容器中注册bean；BeanDefinetion
 *           internalAutoProxyCreator=AnnotationAwareAspectJAutoProxyCreator
 * 
 *        给容器中注册一个AnnotationAwareAspectJAutoProxyCreator；
 * 
 * 2、 AnnotationAwareAspectJAutoProxyCreator：
 *        AnnotationAwareAspectJAutoProxyCreator
 *           ->AspectJAwareAdvisorAutoProxyCreator
 *              ->AbstractAdvisorAutoProxyCreator
 *                 ->AbstractAutoProxyCreator
 *                       implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware
 *                    关注后置处理器（在bean初始化完成前后做事情）、自动装配BeanFactory
 * 
 * AbstractAutoProxyCreator.setBeanFactory()
 * AbstractAutoProxyCreator.有后置处理器的逻辑；
 * 
 * AbstractAdvisorAutoProxyCreator.setBeanFactory()-》initBeanFactory()
 * 
 * AnnotationAwareAspectJAutoProxyCreator.initBeanFactory()
 *
 *
 * 流程：
 *        1）、传入配置类，创建ioc容器
 *        2）、注册配置类，调用refresh（）刷新容器；
 *        3）、registerBeanPostProcessors(beanFactory);注册bean的后置处理器来方便拦截bean的创建；
 *           1）、先获取ioc容器已经定义了的需要创建对象的所有BeanPostProcessor
 *           2）、给容器中加别的BeanPostProcessor
 *           3）、优先注册实现了PriorityOrdered接口的BeanPostProcessor；
 *           4）、再给容器中注册实现了Ordered接口的BeanPostProcessor；
 *           5）、注册没实现优先级接口的BeanPostProcessor；
 *           6）、注册BeanPostProcessor，实际上就是创建BeanPostProcessor对象，保存在容器中；
 *              创建internalAutoProxyCreator的BeanPostProcessor【AnnotationAwareAspectJAutoProxyCreator】
 *              1）、创建Bean的实例
 *              2）、populateBean；给bean的各种属性赋值
 *              3）、initializeBean：初始化bean；
 *                    1）、invokeAwareMethods()：处理Aware接口的方法回调
 *                    2）、applyBeanPostProcessorsBeforeInitialization()：应用后置处理器的postProcessBeforeInitialization（）
 *                    3）、invokeInitMethods()；执行自定义的初始化方法
 *                    4）、applyBeanPostProcessorsAfterInitialization()；执行后置处理器的postProcessAfterInitialization（）；
 *              4）、BeanPostProcessor(AnnotationAwareAspectJAutoProxyCreator)创建成功；--》aspectJAdvisorsBuilder
 *           7）、把BeanPostProcessor注册到BeanFactory中；
 *              beanFactory.addBeanPostProcessor(postProcessor);
 * =======以上是创建和注册AnnotationAwareAspectJAutoProxyCreator的过程========
 * 
 *           AnnotationAwareAspectJAutoProxyCreator => InstantiationAwareBeanPostProcessor
 *        4）、finishBeanFactoryInitialization(beanFactory);完成BeanFactory初始化工作；创建剩下的单实例bean
 *           1）、遍历获取容器中所有的Bean，依次创建对象getBean(beanName);
 *              getBean->doGetBean()->getSingleton()->
 *           2）、创建bean
 *              【AnnotationAwareAspectJAutoProxyCreator在所有bean创建之前会有一个拦截，InstantiationAwareBeanPostProcessor，会调用postProcessBeforeInstantiation()】
 *              1）、先从缓存中获取当前bean，如果能获取到，说明bean是之前被创建过的，直接使用，否则再创建；
 *                 只要创建好的Bean都会被缓存起来
 *              2）、createBean（）;创建bean；
 *                 AnnotationAwareAspectJAutoProxyCreator 会在任何bean创建之前先尝试返回bean的实例
 *                 【BeanPostProcessor是在Bean对象创建完成初始化前后调用的】
 *                 【InstantiationAwareBeanPostProcessor是在创建Bean实例之前先尝试用后置处理器返回对象的】
 *                 1）、resolveBeforeInstantiation(beanName, mbdToUse);解析BeforeInstantiation
 *                    希望后置处理器在此能返回一个代理对象；如果能返回代理对象就使用，如果不能就继续
 *                    1）、后置处理器先尝试返回对象；
 *                       bean = applyBeanPostProcessorsBeforeInstantiation（）：
 *                          拿到所有后置处理器，如果是InstantiationAwareBeanPostProcessor;
 *                          就执行postProcessBeforeInstantiation
 *                       if (bean != null) {
                        bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                     }
 * 
 *                 2）、doCreateBean(beanName, mbdToUse, args);真正的去创建一个bean实例；和3.6流程一样；
 *                 3）、
 *           
 *        
 * AnnotationAwareAspectJAutoProxyCreator【InstantiationAwareBeanPostProcessor】 的作用：
 * 1）、每一个bean创建之前，调用postProcessBeforeInstantiation()；
 *        关心MathCalculator和LogAspect的创建
 *        1）、判断当前bean是否在advisedBeans中（保存了所有需要增强bean）
 *        2）、判断当前bean是否是基础类型的Advice、Pointcut、Advisor、AopInfrastructureBean，
 *           或者是否是切面（@Aspect）
 *        3）、是否需要跳过
 *           1）、获取候选的增强器（切面里面的通知方法）【List<Advisor> candidateAdvisors】
 *              每一个封装的通知方法的增强器是 InstantiationModelAwarePointcutAdvisor；
 *              判断每一个增强器是否是 AspectJPointcutAdvisor 类型的；返回true
 *           2）、永远返回false
 * 
 * 2）、创建对象
 * postProcessAfterInitialization；
 *        return wrapIfNecessary(bean, beanName, cacheKey);//包装如果需要的情况下
 *        1）、获取当前bean的所有增强器（通知方法）  Object[]  specificInterceptors
 *           1、找到候选的所有的增强器（找哪些通知方法是需要切入当前bean方法的）
 *           2、获取到能在bean使用的增强器。
 *           3、给增强器排序
 *        2）、保存当前bean在advisedBeans中；
 *        3）、如果当前bean需要增强，创建当前bean的代理对象；
 *           1）、获取所有增强器（通知方法）
 *           2）、保存到proxyFactory
 *           3）、创建代理对象：Spring自动决定
 *              JdkDynamicAopProxy(config);jdk动态代理；
 *              ObjenesisCglibAopProxy(config);cglib的动态代理；
 *        4）、给容器中返回当前组件使用cglib增强了的代理对象；
 *        5）、以后容器中获取到的就是这个组件的代理对象，执行目标方法的时候，代理对象就会执行通知方法的流程；
 *        
 *     
 *     3）、目标方法执行  ；
 *        容器中保存了组件的代理对象（cglib增强后的对象），这个对象里面保存了详细信息（比如增强器，目标对象，xxx）；
 *        1）、CglibAopProxy.intercept();拦截目标方法的执行
 *        2）、根据ProxyFactory对象获取将要执行的目标方法拦截器链；
 *           List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
 *           1）、List<Object> interceptorList保存所有拦截器 5
 *              一个默认的ExposeInvocationInterceptor 和 4个增强器；
 *           2）、遍历所有的增强器，将其转为Interceptor；
 *              registry.getInterceptors(advisor);
 *           3）、将增强器转为List<MethodInterceptor>；
 *              如果是MethodInterceptor，直接加入到集合中
 *              如果不是，使用AdvisorAdapter将增强器转为MethodInterceptor；
 *              转换完成返回MethodInterceptor数组；
 * 
 *        3）、如果没有拦截器链，直接执行目标方法;
 *           拦截器链（每一个通知方法又被包装为方法拦截器，利用MethodInterceptor机制）
 *        4）、如果有拦截器链，把需要执行的目标对象，目标方法，
 *           拦截器链等信息传入创建一个 CglibMethodInvocation 对象，
 *           并调用 Object retVal =  mi.proceed();
 *        5）、拦截器链的触发过程;
 *           1)、如果没有拦截器执行执行目标方法，或者拦截器的索引和拦截器数组-1大小一样（指定到了最后一个拦截器）执行目标方法；
 *           2)、链式获取每一个拦截器，拦截器执行invoke方法，每一个拦截器等待下一个拦截器执行完成返回以后再来执行；
 *              拦截器链的机制，保证通知方法与目标方法的执行顺序；
 *        
 *     总结：
 *        1）、  @EnableAspectJAutoProxy 开启AOP功能
 *        2）、 @EnableAspectJAutoProxy 会给容器中注册一个组件 AnnotationAwareAspectJAutoProxyCreator
 *        3）、AnnotationAwareAspectJAutoProxyCreator是一个后置处理器；
 *        4）、容器的创建流程：
 *           1）、registerBeanPostProcessors（）注册后置处理器；创建AnnotationAwareAspectJAutoProxyCreator对象
 *           2）、finishBeanFactoryInitialization（）初始化剩下的单实例bean
 *              1）、创建业务逻辑组件和切面组件
 *              2）、AnnotationAwareAspectJAutoProxyCreator拦截组件的创建过程
 *              3）、组件创建完之后，判断组件是否需要增强
 *                 是：切面的通知方法，包装成增强器（Advisor）;给业务逻辑组件创建一个代理对象（cglib）；
 *        5）、执行目标方法：
 *           1）、代理对象执行目标方法
 *           2）、CglibAopProxy.intercept()；
 *              1）、得到目标方法的拦截器链（增强器包装成拦截器MethodInterceptor）
 *              2）、利用拦截器的链式机制，依次进入每一个拦截器进行执行；
 *              3）、效果：
 *                 正常执行：前置通知-》目标方法-》后置通知-》返回通知
 *                 出现异常：前置通知-》目标方法-》后置通知-》异常通知
 *        
 * 
 * 
 */
@EnableAspectJAutoProxy
@Configuration
public class MainConfigOfAOP {
    
   //业务逻辑类加入容器中
   @Bean
   public MathCalculator calculator(){
      return new MathCalculator();
   }

   //切面类加入到容器中
   @Bean
   public LogAspects logAspects(){
      return new LogAspects();
   }
}
```

# 事务

```java
@EnableTransactionManagement
   1.proxyTargetClass = true 说明是cglib代理，false jdk代理
   2.AdviceMode 默认PROXY，有 PROXY及ASPECTJ两种
   3.order多事务时排序
@Transactional 
   1.指定事务管理器
   2.传播机制
   3.隔离级别
   4.超时     该属性用于设置事务的超时秒数，默认值为-1表示永不超时
   5.只读  在将事务设置成只读后，此时若要进行写的操作，会抛异常
   6.回滚类.class，回滚对象
   7.不会滚类.class,对象
```

### 1. 题目

1. 事务失效情况

   1. 数据库引擎不支持事务
   2. springmvc+spring **context:component-scan**重复扫描可能会引起失效
   3. 非public方法 **@Transactional 注解，它不会报错，事务也会失效。**可在自己设置异常类改变默认
   4. 默认情况下，运行时异常才会导致回滚。
   5. @Transactional 注解，优先级顺序为方法>实现类>接口；
   6. 在非jdk动态代理情况下尽量不要在接口上申明事务，注解不能继承，cglib基于类的代理。

2. 隔离级别

   - TransactionDefinition.ISOLATION_DEFAULT：使用后端数据库默认的隔离界别，MySQL默认采用的REPEATABLE_READ隔离级别，Oracle默认采用的READ_COMMITTED隔离级别。
   - TransactionDefinition.ISOLATION_READ_UNCOMMITTED：最低的隔离级别，允许读取，允许读取尚未提交的的数据变更，可能会导致脏读、幻读或不可重复读。
   - TransactionDefinition.ISOLATION_READ_COMMITTED：允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生。
   - TransactionDefinition.ISOLATION_REPEATABLE_READ：对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。
   - TransactionDefinition.ISOLATION_SERIALIZABLE：最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就说，该级别可以阻止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

   

   传播机制

   1>PROPAGATION_REQUIRED: 加入父事务（同时成功失败），没有则开启一个子事务

   2>PROPAGATION_SUPPORTS:加入父事务（同时成功失败），没有事务非事务方式运行。

   3>PROPAGATION_MANDATORY:加入父事务（同时成功失败），假设当前没有事务，就抛出异常。 

   4>PROPAGATION_REQUIRES_NEW:新建事务，假设存在父事务。把父事务挂起。

   5>PROPAGATION_NOT_SUPPORTED:以非事务方式运行操作。假设存在父事务，就把父事务挂起。

   6>PROPAGATION_NEVER:以非事务方式运行，假设当前存在事务，则抛出异常。

   7>PROPAGATION_NESTED:如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与PROPAGATION_REQUIRED类似的操作。

    

   

### 2. 事务分类

 1. 编程式事务   缺点：代码冗余，嵌入业务逻辑；优点： 细粒度

    使用TransactionTemplate 手动执行，回滚

    ```java
    @Autowired
        private TransactionTemplate transactionTemplate;
    
        public void test() {
            //无返回值
            transactionTemplate.execute(new TransactionCallbackWithoutResult() {
                @Override
                public void doInTransactionWithoutResult(TransactionStatus status) {
    
                }
    
            });
    
            //有返回值
            transactionTemplate.execute(new TransactionCallback<String>() {
                @Override
                public String doInTransaction(TransactionStatus status) {
                    return null;
                }
            });
        }
    ```

    

 2. 声明式事务 对正常业务逻辑无影响，但最多支持方法级别事务

```java
package com.atguigu.tx;

import java.beans.PropertyVetoException;

import javax.sql.DataSource;

import org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator;
import org.springframework.aop.framework.autoproxy.InfrastructureAdvisorAutoProxyCreator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.annotation.AnnotationTransactionAttributeSource;
import org.springframework.transaction.annotation.EnableTransactionManagement;
import org.springframework.transaction.annotation.ProxyTransactionManagementConfiguration;
import org.springframework.transaction.annotation.TransactionManagementConfigurationSelector;
import org.springframework.transaction.annotation.Transactional;

import com.mchange.v2.c3p0.ComboPooledDataSource;

/**
 * 声明式事务：
 * 
 * 环境搭建：
 * 1、导入相关依赖
 *        数据源、数据库驱动、Spring-jdbc模块
 * 2、配置数据源、JdbcTemplate（Spring提供的简化数据库操作的工具）操作数据
 * 3、给方法上标注 @Transactional 表示当前方法是一个事务方法；
 * 4、 @EnableTransactionManagement 开启基于注解的事务管理功能；
 *        @EnableXXX
 * 5、配置事务管理器来控制事务;
 *        @Bean
 *        public PlatformTransactionManager transactionManager()
 * 
 * 
 * 原理：
 * 1）、@EnableTransactionManagement
 *           利用TransactionManagementConfigurationSelector给容器中会导入组件
 *           导入两个组件
 *           AutoProxyRegistrar
 *           ProxyTransactionManagementConfiguration
 * 2）、AutoProxyRegistrar：
 *           给容器中注册一个 InfrastructureAdvisorAutoProxyCreator 组件；
 *           InfrastructureAdvisorAutoProxyCreator：？
 *           利用后置处理器机制在对象创建以后，包装对象，返回一个代理对象（增强器），代理对象执行方法利用拦截器链进行调用；
 * 
 * 3）、ProxyTransactionManagementConfiguration 做了什么？
 *           1、给容器中注册事务增强器；
 *              1）、事务增强器要用事务注解的信息，AnnotationTransactionAttributeSource解析事务注解
 *              2）、事务拦截器：
 *                 TransactionInterceptor；保存了事务属性信息，事务管理器；
 *                 他是一个 MethodInterceptor；
 *                 在目标方法执行的时候；
 *                    执行拦截器链；
 *                    事务拦截器：
 *                       1）、先获取事务相关的属性
 *                       2）、再获取PlatformTransactionManager，如果事先没有添加指定任何transactionmanger
 *                          最终会从容器中按照类型获取一个PlatformTransactionManager；
 *                       3）、执行目标方法
 *                          如果异常，获取到事务管理器，利用事务管理回滚操作；
 *                          如果正常，利用事务管理器，提交事务
 *           
 */
@EnableTransactionManagement
@ComponentScan("com.atguigu.tx")
@Configuration
public class TxConfig {
   
   //数据源
   @Bean
   public DataSource dataSource() throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser("root");
      dataSource.setPassword("123456");
      dataSource.setDriverClass("com.mysql.jdbc.Driver");
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/test");
      return dataSource;
   }
   
   //
   @Bean
   public JdbcTemplate jdbcTemplate() throws Exception{
      //Spring对@Configuration类会特殊处理；给容器中加组件的方法，多次调用都只是从容器中找组件
      JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource());
      return jdbcTemplate;
   }
   
   //注册事务管理器在容器中
   @Bean
   public PlatformTransactionManager transactionManager() throws Exception{
      return new DataSourceTransactionManager(dataSource());
   }
   

}
```

# 扩展原理

```java
package com.atguigu.ext;

import java.util.concurrent.Executor;

import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.context.event.SimpleApplicationEventMulticaster;

import com.atguigu.bean.Blue;

/**
 * 扩展原理：
 * BeanPostProcessor：bean后置处理器，bean创建对象初始化前后进行拦截工作的
 * 
 * 1、BeanFactoryPostProcessor：beanFactory的后置处理器；
 *        在BeanFactory标准初始化之后调用，来定制和修改BeanFactory的内容；
 *        所有的bean定义已经保存加载到beanFactory，但是bean的实例还未创建
 * 
 * 
 * BeanFactoryPostProcessor原理:
 * 1)、ioc容器创建对象
 * 2)、invokeBeanFactoryPostProcessors(beanFactory);
 *        如何找到所有的BeanFactoryPostProcessor并执行他们的方法；
 *           1）、直接在BeanFactory中找到所有类型是BeanFactoryPostProcessor的组件，并执行他们的方法
 *           2）、在初始化创建其他组件前面执行
 * 
 * 2、BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor
 *        postProcessBeanDefinitionRegistry();
 *        在所有bean定义信息将要被加载，bean实例还未创建的；
 * 
 *        优先于BeanFactoryPostProcessor执行；
 *        利用BeanDefinitionRegistryPostProcessor给容器中再额外添加一些组件；
 * 
 *     原理：
 *        1）、ioc创建对象
 *        2）、refresh()-》invokeBeanFactoryPostProcessors(beanFactory);
 *        3）、从容器中获取到所有的BeanDefinitionRegistryPostProcessor组件。
 *           1、依次触发所有的postProcessBeanDefinitionRegistry()方法
 *           2、再来触发postProcessBeanFactory()方法BeanFactoryPostProcessor；
 * 
 *        4）、再来从容器中找到BeanFactoryPostProcessor组件；然后依次触发postProcessBeanFactory()方法
 *     
 * 3、ApplicationListener：监听容器中发布的事件。事件驱动模型开发；
 *       public interface ApplicationListener<E extends ApplicationEvent>
 *        监听 ApplicationEvent 及其下面的子事件；
 * 
 *      步骤：
 *        1）、写一个监听器（ApplicationListener实现类）来监听某个事件（ApplicationEvent及其子类）
 *           @EventListener;
 *           原理：使用EventListenerMethodProcessor处理器来解析方法上的@EventListener；
 * 
 *        2）、把监听器加入到容器；
 *        3）、只要容器中有相关事件的发布，我们就能监听到这个事件；
 *              ContextRefreshedEvent：容器刷新完成（所有bean都完全创建）会发布这个事件；
 *              ContextClosedEvent：关闭容器会发布这个事件；
 *        4）、发布一个事件：
 *              applicationContext.publishEvent()；
 *     
 *  原理：
 *     ContextRefreshedEvent、IOCTest_Ext$1[source=我发布的时间]、ContextClosedEvent；
 *  1）、ContextRefreshedEvent事件：
 *     1）、容器创建对象：refresh()；
 *     2）、finishRefresh();容器刷新完成会发布ContextRefreshedEvent事件
 *  2）、自己发布事件；
 *  3）、容器关闭会发布ContextClosedEvent；
 *  
 *  【事件发布流程】：
 *     3）、publishEvent(new ContextRefreshedEvent(this));
 *           1）、获取事件的多播器（派发器）：getApplicationEventMulticaster()
 *           2）、multicastEvent派发事件：
 *           3）、获取到所有的ApplicationListener；
 *              for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
 *              1）、如果有Executor，可以支持使用Executor进行异步派发；
 *                 Executor executor = getTaskExecutor();
 *              2）、否则，同步的方式直接执行listener方法；invokeListener(listener, event);
 *               拿到listener回调onApplicationEvent方法；
 *  
 *  【事件多播器（派发器）】
 *     1）、容器创建对象：refresh();
 *     2）、initApplicationEventMulticaster();初始化ApplicationEventMulticaster；
 *        1）、先去容器中找有没有id=“applicationEventMulticaster”的组件；
 *        2）、如果没有this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
 *           并且加入到容器中，我们就可以在其他组件要派发事件，自动注入这个applicationEventMulticaster；
 *  
 *  【容器中有哪些监听器】
 *     1）、容器创建对象：refresh();
 *     2）、registerListeners();
 *        从容器中拿到所有的监听器，把他们注册到applicationEventMulticaster中；
 *        String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
 *        //将listener注册到ApplicationEventMulticaster中
 *        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
 *        
 *   SmartInitializingSingleton 原理：->afterSingletonsInstantiated();
 *        1）、ioc容器创建对象并refresh()；
 *        2）、finishBeanFactoryInitialization(beanFactory);初始化剩下的单实例bean；
 *           1）、先创建所有的单实例bean；getBean();
 *           2）、获取所有创建好的单实例bean，判断是否是SmartInitializingSingleton类型的；
 *              如果是就调用afterSingletonsInstantiated();
 *        
 * 
 *
 */
@ComponentScan("com.atguigu.ext")
@Configuration
public class ExtConfig {
   
   @Bean
   public Blue blue(){
      return new Blue();
   }

}
```

# spring启动源码

```
Spring容器的refresh()【创建刷新】;
1、prepareRefresh()刷新前的预处理;
   1）、initPropertySources()初始化一些属性设置;子类自定义个性化的属性设置方法；
   2）、getEnvironment().validateRequiredProperties();检验属性的合法等
   3）、earlyApplicationEvents= new LinkedHashSet<ApplicationEvent>();保存容器中的一些早期的事件；
2、obtainFreshBeanFactory();获取BeanFactory；
   1）、refreshBeanFactory();刷新【创建】BeanFactory；
         创建了一个this.beanFactory = new DefaultListableBeanFactory();
         设置id；
   2）、getBeanFactory();返回刚才GenericApplicationContext创建的BeanFactory对象；
   3）、将创建的BeanFactory【DefaultListableBeanFactory】返回；
3、prepareBeanFactory(beanFactory);BeanFactory的预准备工作（BeanFactory进行一些设置）；
   1）、设置BeanFactory的类加载器、支持表达式解析器...
   2）、添加部分BeanPostProcessor【ApplicationContextAwareProcessor】
   3）、设置忽略的自动装配的接口EnvironmentAware、EmbeddedValueResolverAware、xxx；
   4）、注册可以解析的自动装配；我们能直接在任何组件中自动注入：
         BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext
   5）、添加BeanPostProcessor【ApplicationListenerDetector】
   6）、添加编译时的AspectJ；
   7）、给BeanFactory中注册一些能用的组件；
      environment【ConfigurableEnvironment】、
      systemProperties【Map<String, Object>】、
      systemEnvironment【Map<String, Object>】
4、postProcessBeanFactory(beanFactory);BeanFactory准备工作完成后进行的后置处理工作；
   1）、子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置
======================以上是BeanFactory的创建及预准备工作==================================
5、invokeBeanFactoryPostProcessors(beanFactory);执行BeanFactoryPostProcessor的方法；
   BeanFactoryPostProcessor：BeanFactory的后置处理器。在BeanFactory标准初始化之后执行的；
   两个接口：BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor
   1）、执行BeanFactoryPostProcessor的方法；
      先执行BeanDefinitionRegistryPostProcessor
      1）、获取所有的BeanDefinitionRegistryPostProcessor；
      2）、看先执行实现了PriorityOrdered优先级接口的BeanDefinitionRegistryPostProcessor、
         postProcessor.postProcessBeanDefinitionRegistry(registry)
      3）、在执行实现了Ordered顺序接口的BeanDefinitionRegistryPostProcessor；
         postProcessor.postProcessBeanDefinitionRegistry(registry)
      4）、最后执行没有实现任何优先级或者是顺序接口的BeanDefinitionRegistryPostProcessors；
         postProcessor.postProcessBeanDefinitionRegistry(registry)
         
      
      再执行BeanFactoryPostProcessor的方法
      1）、获取所有的BeanFactoryPostProcessor
      2）、看先执行实现了PriorityOrdered优先级接口的BeanFactoryPostProcessor、
         postProcessor.postProcessBeanFactory()
      3）、在执行实现了Ordered顺序接口的BeanFactoryPostProcessor；
         postProcessor.postProcessBeanFactory()
      4）、最后执行没有实现任何优先级或者是顺序接口的BeanFactoryPostProcessor；
         postProcessor.postProcessBeanFactory()
6、registerBeanPostProcessors(beanFactory);注册BeanPostProcessor（Bean的后置处理器）【 intercept bean creation】
      不同接口类型的BeanPostProcessor；在Bean创建前后的执行时机是不一样的
      BeanPostProcessor、
      DestructionAwareBeanPostProcessor、
      InstantiationAwareBeanPostProcessor、
      SmartInstantiationAwareBeanPostProcessor、
      MergedBeanDefinitionPostProcessor【internalPostProcessors】、
      
      1）、获取所有的 BeanPostProcessor;后置处理器都默认可以通过PriorityOrdered、Ordered接口来执行优先级
      2）、先注册PriorityOrdered优先级接口的BeanPostProcessor；
         把每一个BeanPostProcessor；添加到BeanFactory中
         beanFactory.addBeanPostProcessor(postProcessor);
      3）、再注册Ordered接口的
      4）、最后注册没有实现任何优先级接口的
      5）、最终注册MergedBeanDefinitionPostProcessor；
      6）、注册一个ApplicationListenerDetector；来在Bean创建完成后检查是否是ApplicationListener，如果是
         applicationContext.addApplicationListener((ApplicationListener<?>) bean);
7、initMessageSource();初始化MessageSource组件（做国际化功能；消息绑定，消息解析）；
      1）、获取BeanFactory
      2）、看容器中是否有id为messageSource的，类型是MessageSource的组件
         如果有赋值给messageSource，如果没有自己创建一个DelegatingMessageSource；
            MessageSource：取出国际化配置文件中的某个key的值；能按照区域信息获取；
      3）、把创建好的MessageSource注册在容器中，以后获取国际化配置文件的值的时候，可以自动注入MessageSource；
         beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);   
         MessageSource.getMessage(String code, Object[] args, String defaultMessage, Locale locale);
8、initApplicationEventMulticaster();初始化事件派发器；
      1）、获取BeanFactory
      2）、从BeanFactory中获取applicationEventMulticaster的ApplicationEventMulticaster；
      3）、如果上一步没有配置；创建一个SimpleApplicationEventMulticaster
      4）、将创建的ApplicationEventMulticaster添加到BeanFactory中，以后其他组件直接自动注入
9、onRefresh();留给子容器（子类）
      1、子类重写这个方法，在容器刷新的时候可以自定义逻辑；
10、registerListeners();给容器中将所有项目里面的ApplicationListener注册进来；
      1、从容器中拿到所有的ApplicationListener
      2、将每个监听器添加到事件派发器中；
         getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
      3、派发之前步骤产生的事件；
11、finishBeanFactoryInitialization(beanFactory);初始化所有剩下的单实例bean；
   1、beanFactory.preInstantiateSingletons();初始化后剩下的单实例bean
      1）、获取容器中的所有Bean，依次进行初始化和创建对象
      2）、获取Bean的定义信息；RootBeanDefinition
      3）、Bean不是抽象的，是单实例的，是懒加载；
         1）、判断是否是FactoryBean；是否是实现FactoryBean接口的Bean；
         2）、不是工厂Bean。利用getBean(beanName);创建对象
            0、getBean(beanName)； ioc.getBean();
            1、doGetBean(name, null, null, false);
            2、先获取缓存中保存的单实例Bean。如果能获取到说明这个Bean之前被创建过（所有创建过的单实例Bean都会被缓存起来）
               从private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);获取的
            3、缓存中获取不到，开始Bean的创建对象流程；
            4、标记当前bean已经被创建
            5、获取Bean的定义信息；
            6、【获取当前Bean依赖的其他Bean;如果有按照getBean()把依赖的Bean先创建出来；】
            7、启动单实例Bean的创建流程；
               1）、createBean(beanName, mbd, args);
               2）、Object bean = resolveBeforeInstantiation(beanName, mbdToUse);让BeanPostProcessor先拦截返回代理对象；
                  【InstantiationAwareBeanPostProcessor】：提前执行；
                  先触发：postProcessBeforeInstantiation()；
                  如果有返回值：触发postProcessAfterInitialization()；
               3）、如果前面的InstantiationAwareBeanPostProcessor没有返回代理对象；调用4）
               4）、Object beanInstance = doCreateBean(beanName, mbdToUse, args);创建Bean
                   1）、【创建Bean实例】；createBeanInstance(beanName, mbd, args);
                     利用工厂方法或者对象的构造器创建出Bean实例；
                   2）、applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                     调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition(mbd, beanType, beanName);
                   3）、【Bean属性赋值】populateBean(beanName, mbd, instanceWrapper);
                     赋值之前：
                     1）、拿到InstantiationAwareBeanPostProcessor后置处理器；
                        postProcessAfterInstantiation()；
                     2）、拿到InstantiationAwareBeanPostProcessor后置处理器；
                        postProcessPropertyValues()；
                     =====赋值之前：===
                     3）、应用Bean属性的值；为属性利用setter方法等进行赋值；
                        applyPropertyValues(beanName, mbd, bw, pvs);
                   4）、【Bean初始化】initializeBean(beanName, exposedObject, mbd);
                     1）、【执行Aware接口方法】invokeAwareMethods(beanName, bean);执行xxxAware接口的方法
                        BeanNameAware\BeanClassLoaderAware\BeanFactoryAware
                     2）、【执行后置处理器初始化之前】applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
                        BeanPostProcessor.postProcessBeforeInitialization（）;
                     3）、【执行初始化方法】invokeInitMethods(beanName, wrappedBean, mbd);
                        1）、是否是InitializingBean接口的实现；执行接口规定的初始化；
                        2）、是否自定义初始化方法；
                     4）、【执行后置处理器初始化之后】applyBeanPostProcessorsAfterInitialization
                        BeanPostProcessor.postProcessAfterInitialization()；
                   5）、注册Bean的销毁方法；
               5）、将创建的Bean添加到缓存中singletonObjects；
            ioc容器就是这些Map；很多的Map里面保存了单实例Bean，环境信息。。。。；
      所有Bean都利用getBean创建完成以后；
         检查所有的Bean是否是SmartInitializingSingleton接口的；如果是；就执行afterSingletonsInstantiated()；
12、finishRefresh();完成BeanFactory的初始化创建工作；IOC容器就创建完成；
      1）、initLifecycleProcessor();初始化和生命周期有关的后置处理器；LifecycleProcessor
         默认从容器中找是否有lifecycleProcessor的组件【LifecycleProcessor】；如果没有new DefaultLifecycleProcessor();
         加入到容器；
         
         写一个LifecycleProcessor的实现类，可以在BeanFactory
            void onRefresh();
            void onClose();    
      2）、    getLifecycleProcessor().onRefresh();
         拿到前面定义的生命周期处理器（BeanFactory）；回调onRefresh()；
      3）、publishEvent(new ContextRefreshedEvent(this));发布容器刷新完成事件；
      4）、liveBeansView.registerApplicationContext(this);
   
   ======总结===========
   1）、Spring容器在启动的时候，先会保存所有注册进来的Bean的定义信息；
      1）、xml注册bean；<bean>
      2）、注解注册Bean；@Service、@Component、@Bean、xxx
   2）、Spring容器会合适的时机创建这些Bean
      1）、用到这个bean的时候；利用getBean创建bean；创建好以后保存在容器中；
      2）、统一创建剩下所有的bean的时候；finishBeanFactoryInitialization()；
   3）、后置处理器；BeanPostProcessor
      1）、每一个bean创建完成，都会使用各种后置处理器进行处理；来增强bean的功能；
         AutowiredAnnotationBeanPostProcessor:处理自动注入
         AnnotationAwareAspectJAutoProxyCreator:来做AOP功能；
         xxx....
         增强的功能注解：
         AsyncAnnotationBeanPostProcessor
         ....
   4）、事件驱动模型；
      ApplicationListener；事件监听；
      ApplicationEventMulticaster；事件派发：
   
```