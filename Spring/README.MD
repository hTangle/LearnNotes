## Spring MVC
### 组成：DipatcherServlet，处理器映射，处理器，视图解析器，视图
1. 用户发送请求由前置控制器DispatcherServlet来决定哪一个页面的控制器进行处理并把请求委托给它，
2. 在由HandlerMapping将请求映射为HandlerExecutionChain对象(包含Handler处理器对象（页面控制器），多个HandlerInterceptor对象即拦截器)，
3. 再返回给DispatcherServlet，DispatcherServlet再次发送请求给HandlerAdapter，
4. HandlerAdapter将处理器包装为适配器，调用处理器相应功能处理方法，Handler返回ModelAnView给HandlerAdapter，
5. HandlerAdapter发送给DispatcherServlet进行视图的解析（ViewResolver），
6. ViewResolver将逻辑视图解析为具体的视图，返回给DispatcherServlet，
7. 在进行视图的渲染（View），返回给DispatcherServlet，最后通过DispatcherServlet将视图返回给用户。

## Spring注解
* @controller控制器（注入服务）
* @RestController：相当于@ResponseBody+@Controller结合，返回JSON需要在对应的方法上加@ResponseBody
* @service 服务（注入dao）
* @repository dao（实现dao访问）
* @component （把普通的pojo实例化到spring容器中，相当于配置文件中<bean id="" class="">）
* @EnableAutoConfiguration：借助AutoConfigurationImportSelector.class（SpringFactoriesLoader.loadFactoryNames()）可以帮助SpringBoot应用将所有符合条件的@Configuration配置都加载到当前SpringBoot创建并使用的IoC容器中。
*	SpringFactoriesLoader：
*	从META-INF/spring-factories加载配置，有三个静态方法：loadFactories：加载指定的factoryClass并进行实例化。loadFactoryNames：加载指定的factoryClass的名称集合。instantiateFactory：对指定的factoryClass进行实例化。

### @Autowired和@Resource的区别
* 都可以用来装配bean，都可以写在setter方法上
* @Autowired默认按类型装配，默认情况下必须要求依赖对象存在，如果要允许null值，设置require属性为false：@Autowired(required=false)，想要按照名称装配，接口@Qualifier使用
* @Resource注解输入J2EE，默认按照名称装配，名称可以通过name属性指定，找不到名称才按照类型装配

Spring AOP
* 基于动态代理来实现，如果使用接口的，用JDK提供的动态代理实现，如果没有接口，则使用CGLIB来实现
* Spring AOP 需要依赖IOC容器来管理
* Spring AOP 会在 IOC 容器创建 bean 实例的最后对 bean 进行处理。其实就是在这一步进行代理增强
* 如果被代理的目标类实现了一个或多个自定义的接口，那么就会使用 JDK 动态代理，如果没有实现任何接口，会使用 CGLIB 实现代理，如果设置了 proxy-target-class="true"，那么都会使用 CGLIB。
* JDK 动态代理基于接口，所以只有接口中的方法会被增强，而 CGLIB 基于类继承，需要注意就是如果方法使用了 final 修饰，或者是 private 方法，是不能被增强的。
### BeanPostProcessor有两个方法
* postProcessBeforeInitialization
* postProcessAfterInitialization
> 在执行init-method前后会调用以上两个方法
> 在创建bean之后进行动态代理增强

Bean的生命周期：由IoC容器控制
1）实例化bean对象（通过构造方法或工厂方法）
2）设置对象属性（依赖注入）
3）检查Aware相关接口并设置相关依赖
4）BeanPostProcessor前置处理
5）检查是否是InitializingBean以决定是否调用AfterPropertiesSet方法
6）检查是否配置有自定义的init-method
7）BeanPostProcessor后置处理
8）注册必要的Destruction相关回调接口
9）使用中
10）是否实现DisposableBean接口
11）是否配置有自定义destroy方法
BeanFactory和ApplicationContext
ApplicationContext提供更多的拓展功能，初始化即加载，自动注册，MessageSource提供国际化消息访问，资源访问，事件传播
BeanFactory延迟加载，手动注册，不支持AOP
