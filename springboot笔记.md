# SpringBoot学习笔记

## 1.@Value和@ConfigurationProperties对比

|                 | @value     | @ConfigurationProperties（属性值需要set方法） |
| --------------- | ---------- | --------------------------------------------- |
| 功能            | 获取单个值 | 自动生成值                                    |
| 松散原因        | 不知驼峰   | 支持                                          |
| SpEL（#{}）     | 支持       | 不支持                                        |
| jsr303          | 不支持     | 支持类注解@Validated属性注解@Email            |
| 复杂类型（map） | 不支持     | 支持                                          |

## 2.@propertySource &  @ImportResource

### @propertySource(加载制定配置文件与@ConfigurationProperties联合使用)

### @ImportResource加载spring的beans.xml配置文件，让里面的内容生效



## 3. SpringBoot 过滤器, 拦截器, 监听器 对比及使用场景 原

![img](https://oscimg.oschina.net/oscnet/03c84353cbcfc57dddf7714cba62cfed662.jpg)





![img](https://oscimg.oschina.net/oscnet/1090617-20180515204018593-1889287518.png)

**1. 过滤器 (实现 javax.servlet.Filter 接口)**

>  ① 过滤器是在web应用启动的时候初始化一次, 在web应用停止的时候销毁. ② 可以对请求的URL进行过滤, 对敏感词过滤,  ③ 挡在拦截器的外层 ④ Filter 是 Servlet 规范的一部分 

**2. 拦截器 (实现 org.springframework.web.servlet.HandlerInterceptor 接口)**

>  ① 不依赖Spring容器, 可以使用 Spring 容器管理的Bean ② 拦截器通过动态代理进行 ③ 拦截器应用场景, 性能分析, 权限检查, 日志记录 

**3. 监听器 (实现 javax.servlet.ServletRequestListener, javax.servlet.http.HttpSessionListener, javax.servlet.ServletContextListener 等等接口)**

>  ① 主要用来监听对象的创建与销毁的发生, 比如 session 的创建销毁, request 的创建销毁, ServletContext 创建销毁 