# springmvc注解大全

@ModelAttribute

@ControllerAdvice

@ExceptionHandler

@InitBinder

## 流程

<img src="C:\Users\Arpgalaxy\AppData\Roaming\Typora\typora-user-images\image-20200530193448020.png" alt="image-20200530193448020" style="zoom:200%;" />

# servlet3.0原理



```java
1、Servlet容器启动会扫描，当前应用里面每一个jar包的
   ServletContainerInitializer的实现
2、提供ServletContainerInitializer的实现类；
   必须绑定在，META-INF/services/javax.servlet.ServletContainerInitializer
   文件的内容就是ServletContainerInitializer实现类的全类名；

总结：容器在启动应用的时候，会扫描当前应用每一个jar包里面
META-INF/services/javax.servlet.ServletContainer   Initializer
指定的实现类，启动并运行这个实现类的方法；传入感兴趣的类型；


ServletContainerInitializer；
@HandlesTypes；
//容器启动的时候会将@HandlesTypes指定的这个类型下面的子类（实现类，子接口等）传递过来；
//传入感兴趣的类型；
@HandlesTypes(value={HelloService.class})
public class MyServletContainerInitializer implements ServletContainerInitializer {

	/**
	 * 应用启动的时候，会运行onStartup方法；
	 * 
	 * Set<Class<?>> arg0：感兴趣的类型的所有子类型；
	 * ServletContext arg1:代表当前Web应用的ServletContext；一个Web应用一个ServletContext；
	 * 
	 * 1）、使用ServletContext注册Web组件（Servlet、Filter、Listener）
	 * 2）、使用编码的方式，在项目启动的时候给ServletContext里面添加组件；
	 * 		必须在项目启动的时候来添加；
	 * 		1）、ServletContainerInitializer得到的ServletContext；
	 * 		2）、ServletContextListener得到的ServletContext；
	 */
	@Override
	public void onStartup(Set<Class<?>> arg0, ServletContext sc) throws ServletException {
		// TODO Auto-generated method stub
		System.out.println("感兴趣的类型：");
		for (Class<?> claz : arg0) {
			System.out.println(claz);
		}
		
		//注册组件  ServletRegistration  
		ServletRegistration.Dynamic servlet = sc.addServlet("userServlet", new UserServlet());
		//配置servlet的映射信息
		servlet.addMapping("/user");
		
		
		//注册Listener
		sc.addListener(UserListener.class);
		
		//注册Filter  FilterRegistration
		FilterRegistration.Dynamic filter = sc.addFilter("userFilter", UserFilter.class);
		//配置Filter的映射信息
		filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, "/*");
		
	}

}

```

# springmvc 启动原理

```
1、web容器在启动的时候，会扫描每个jar包下的META-INF/services/javax.servlet.ServletContainerInitializer
2、加载这个文件指定的类SpringServletContainerInitializer
3、spring的应用一启动会加载感兴趣的WebApplicationInitializer接口的下的所有组件；
4、并且为WebApplicationInitializer组件创建对象（组件不是接口，不是抽象类）
   1）、AbstractContextLoaderInitializer：创建根容器；createRootApplicationContext()；
   2）、AbstractDispatcherServletInitializer：
         创建一个web的ioc容器；createServletApplicationContext();
         创建了DispatcherServlet；createDispatcherServlet()；
         将创建的DispatcherServlet添加到ServletContext中；
            getServletMappings();
   3）、AbstractAnnotationConfigDispatcherServletInitializer：注解方式配置的DispatcherServlet初始化器
         创建根容器：createRootApplicationContext()
               getRootConfigClasses();传入一个配置类
         创建web的ioc容器： createServletApplicationContext();
               获取配置类；getServletConfigClasses();
   
总结：
   以注解方式来启动SpringMVC；继承AbstractAnnotationConfigDispatcherServletInitializer；
实现抽象方法指定DispatcherServlet的配置信息；

===========================
定制SpringMVC；
1）、@EnableWebMvc:开启SpringMVC定制配置功能；
   <mvc:annotation-driven/>；

2）、配置组件（视图解析器、视图映射、静态资源映射、拦截器。。。）
   extends WebMvcConfigurerAdapter（废弃）
   extends WebMvcConfigurer    
   
```