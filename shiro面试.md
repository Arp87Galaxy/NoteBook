# shiro面试

shiro拦截器处理链执行顺序

![在这里插入图片描述](C:\Users\Arpgalaxy\Desktop\md文件\images\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MDAwNzg5,size_16,color_FFFFFF,t_70)

A1：根据到spring容器查找实现bean，并根据配置把相应的url请求委托给它去执行。

A21：SpringShiroFilter是ShiroFilterFactoryBean的内部类，会执行doFilter()方法。A21没有重写父类的doFilter方法，因此会去寻找父类中的实现，最终调用OncePerRequestFilter(A23)的该方法。

A23：执行doFilter方法，调用doFilterInternal()方法。

A31：doFilterInternal()方法，OncePerRequestFilter的抽象方法，会去子类中寻找它的实现，最终在子类AbstractShiroFilter(A32)找到实现并执行。

B11:AbstractShiroFilter执行doFilterInternal()方法后会调用excuteChain()里面的chain.doFilter()

B12：AbstractShiroFilter(B11)方法中的chain.doFilter()会调用它的(B12)doFilter()。它没有重写父类的doFilter方法，因此会去寻找父类中的实现，最终调用OncePerRequestFilter(B18)的该方法。这个B12可以是表单拦截器，也可以是其它拦截器实现。

B21：doFilterInternal()方法，OncePerRequestFilter的抽象方法，会去子类中寻找它的实现，最终在子类AdviceFilter(B22)找到实现并执行。

B31：AdviceFilter执行doFilterInternal()方法后调用，会调用preHandle()方法，该方法在子类PathMatchingFilter(B32)中实现。

B33:PathMatchingFilter执行preHandle()时调用了onpreHandle()方法。

onpreHandle随后会继续调用其它方法… 最终调用onAccessDenied方法，该方法在子类B99中实现了

onPreHandle主要流程：

1、首先判断是否已经登录过了，如果已经登录过了继续拦截器链即可；

2、如果没有登录，看看是否是登录请求，如果是get方法的登录页面请求，则继续拦截器链（到请求页面），否则如果是get方法的其他页面请求则保存当前请求并重定向到登录页面；

3、如果是post方法的登录页面表单提交请求，则收集用户名/密码登录即可，如果失败了保存错误消息到“shiroLoginFailure”并返回到登录页面；

4、如果登录成功了，且之前有保存的请求，则重定向到之前的这个请求，否则到默认的成功页面。