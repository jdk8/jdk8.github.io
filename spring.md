# Spring

## Spring事务

事务传播机制：https://segmentfault.com/a/1190000020386113

## 常用注解

**Profile**

被Profile注释的Component只有当注释的值(value)与spring.profiles.active的值相同时才会生效。

**ActiveProfiles**

注释在Spring Boot单测类上，如：@ActiveProfiles("test")。

**RunWith(SpringRunner.class)+SpringBootTest**

注释在Spring Boot单测类上，等同于启动了Spring Boot应用。

**Autowired，Qualifier和Resource**

Autowired只能按照type方式注入，按照如下方式注入接口的实现类。

```java
Component("implA")
class ImplA implements InterfaceX {
}

Component("implB")
class ImplB implements InterfaceX {
}

@Autowired
@Qualifier("implB")
InterfaceX interfaceX;
```

Resource注解可以执行通过name和type进行装配，Autowired注解只能通过type进行装配，可以配合Qualifier注解通过name进行装配。参考：https://www.cnblogs.com/think-in-java/p/5474740.html、https://blog.csdn.net/Weixiaohuai/article/details/120853683

**Before，After，Test**

Test是注释在测试方法上，每个测试方法执行之前执行Before方法，执行之后执行After方法。

**Configuration和Bean**

Configuration注释在类上，表示这个类的方法(用Bean注释)返回的对象可以作为Spring的bean。使用方法如下：

```java
public class MyBean {
    public void destroy() {
        System.out.println("my bean destroy");
    }

    public void init() {
        System.out.println("my bean init");
    }

    public MyBean() {
        System.out.println("my bean construct");
    }
}

@Configuration
@ConfigurationProperties(prefix="xxx")
class AppConfig {
    
    @Bean
    public int para() {
        return 1;
    }
    
    @Bean
    public AnotherBean ab() {
        return new AnotherBean();
    }

    // initMethod表示init方法将在MyBean构造函数之后执行
    @Bean(initMethod = "init")
    // 在Bean注释的方法中的参数，按照入参的名称去找对应的bean
    public MyBean myBean(int para, AnotherBean ab) {
        return new MyBean();
    }
}
```

Component和Configuration的区别是：Configuration对Bean方法拦截，使其返回的都是同一个对象，举例如下：

```java
@Data
public class MyBean2 {
    private MyBean myBean;
}

@Component
public class AppConfig {

    @Bean(initMethod = "init")
    MyBean myBean() {
        return new MyBean();
    }

    @Bean
    MyBean2 myBean2() {
        MyBean2 myBean2 = new MyBean2();
        myBean2.setMyBean(myBean());
        return myBean2;
    }
}
```

用Component会构造两个MyBean对象，用Configuration会构造一个MyBean对象。

**ComponentScan**

和Configuration注解一起使用，替代spring xml配置中的component-scan标签，指定spring扫描component路径。

**基于配置初始化属性**

通过ConfigurationProperties、 EnableConfigurationProperties和ConfigurationPropertiesScan给Bean基于配置自动注入属性：https://blog.csdn.net/niugang0920/article/details/115354613

**基于条件构建Bean**

ConditionalOnBean：当给定的在bean存在时,则实例化当前Bean

ConditionalOnMissingBean：当给定的在bean不存在时,则实例化当前Bean

ConditionalOnClass：当给定的类名在类路径上存在，则实例化当前Bean

ConditionalOnMissingClass：当给定的类名在类路径上不存在，则实例化当前Bean

## AOP

### JDK动态代理

实现一个invocationHandler，对接口中的方法进行增强，再根据这个invocationHandler和接口class生成一个proxy对象，调用proxy对象的方法就会调用invocationHandler的invoke方法，代码如下：

```java
interface Person {
    void walk();
}

class Student implements Person {
    @Override
    public void walk() {
        System.out.println("student walk");
    }
}

class PersonInvocationHandler implements InvocationHandler {
    private Person person;
    public PersonInvocationHandler(Person person) {
        this.person = person;
    }

    @Override
    public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
        System.out.println("begin");
        method.invoke(person);
        System.out.println("end");
        return null;
    }
}

// 调用
Person person = new Student();
// 这里也可以不传Student实例
PersonInvocationHandler personInvocationHandler = new PersonInvocationHandler(person);
Person personProxy = (Person) Proxy.newProxyInstance(Person.class.getClassLoader(),
                person.getClass().getInterfaces(), personInvocationHandler);
personProxy.walk();

/* 
输出：
begin
student walk
end
*/
```

**使用动态代理的例子**：Mybatis

Mybatis中的MapperProxy实现了InvocationHandler，每个Mybatis的接口均会生成对应的MapperProxy，再用这个MapperProxy生成Proxy对象，再调用Proxy对象的insert、update等方法。最终调用到MapperProxy的invoke方法。

### CGLIB动态代理

实现MethodInterceptor接口，对被代理方法进行加工。用被代理类和实现MethodInterceptor接口的类对象构造Enhancer对象，用该对象创建被代理对象。GCLIB原理是生成被代理类的子类，来增强被代理类。举例如下：

```java
public class PersonMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before walk");
        return methodProxy.invokeSuper(o, objects);
    }
}

// 调用
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(Student.class);
enhancer.setCallback(new PersonMethodInterceptor());
Student student = (Student) enhancer.create();
student.walk();

/*
输出：
before walk
student walk
*/
```

**使用CGLIB代理的例子**：spring的CglibAopProxy

每个被Spring AOP关联的对象，都会被生成它的代理类，在某些情况下，是通过CGLIB来生成代理类的，CglibAopProxy.DynamicAdvisedInterceptor实现了MethodInterceptor，在intercept方法里，通过责任链模式最终调用到切面的逻辑，比如：ExposeInvocationInterceptor(interceptor)->AspectJAfterAdvice(aspectj)

### Spring AOP

注释在Spring Component类上，表明这个类的方法可以用来增强指定方法，即AOP。

```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;

// AOP类
@Component
@Aspect
public class LogAdvice {

    @Before("within(com.xxx...*) && @annotation(log)")
    public void beforeLog(JoinPoint joinPoint, Log log) {
        System.out.println("method name=" + joinPoint.getSignature().getName());
        System.out.println("args=" + joinPoint.getArgs()[0]);
        System.out.println("before log, annotation info=" + log.detail());
    }

    @AfterReturning("within(com.xxx...*) && @annotation(log)")
    public void afterLog(JoinPoint joinPoint, Log log) {
        System.out.println("after log, annotation info=" + log.detail());
    }

    @AfterThrowing(value = "within(com.xxx...*) && @annotation(log)", throwing = "ex")
    public void afterLogThrowing(JoinPoint joinPoint, Log log, Exception ex) {
        System.out.println("after log throwing, annotation info=" + log.detail());
    }

    @Around(value = "within(com.xxx..*) && @annotation(log)")
    public void aroundLog(ProceedingJoinPoint proceedingJoinPoint, Log log) throws Throwable {
        System.out.println("around log , annotation info=" + log.detail());
        System.out.println("proceedingJoinPoint method name=" + proceedingJoinPoint.getSignature().getName());
        System.out.println("proceedingJoinPoint args=" + proceedingJoinPoint.getArgs()[0]);
        proceedingJoinPoint.proceed();
    }
    
    @Around("execution(* com.xxx.ClassName.methodName(..))")
    public void around1(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
    }
    
    // 定义一个切面
    @PointCut("execution(* com.xxx.ClassName.methodName(..))")
    public void pc() {
    }
    
    // 使用切面
    @Around("pc()")
    public void around2(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
    }
}

// 增强被Log注释的方法
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Log {
    String detail();
}

@Component
public class LogAnnotationTest {

    @Log(detail = "hello world")
    public void testMethod(String msg) {
        System.out.println("test method, msg=" + msg);
    }
}

// 调用
logAnnotationTest.testMethod("this is msg");

/*
输出：
around log , annotation info=hello world
proceedingJoinPoint method name=testMethod
proceedingJoinPoint args=this is msg
method name=testMethod
args=this is msg
before log, annotation info=hello world
test method, msg=this is msg
after log, annotation info=hello world
*/
```

Spring AOP几种无法执行切面的情况：

1.调用自身的其他增强方法

```java
@Component
public class TestServiceImpl implements TestService {

    @Override
    public void callA() {
        System.out.println("call a");
        callB(); // callB无法执行切面，因为这个对象已经变成代理对象了，执行切面需要另外一个代理。
    }

    @Override
    public void callB() {
        System.out.println("call b");
    }

    public void callC() {
        callB();// callB无法执行切面
    }
}

ConfigurableApplicationContext ctx = SpringApplication.run(MySpringBootApplication.class, args);
TestServiceImpl testService = ctx.getBean(TestServiceImpl.class);
testService.callA();
testService.callC();
```

根据代理对象获取被代理对象：

```java
if (AopUtils.isCglibProxy(proxyClass)) {
    Field h = currClass.getDeclaredField("CGLIB$CALLBACK_0");
    h.setAccessible(true);
    Object dynamicAdvisedInterceptor = h.get(proxyClass);
    Field advised = dynamicAdvisedInterceptor.getClass().getDeclaredField("advised");
    advised.setAccessible(true);
    Object target = ((AdvisedSupport) advised.get(dynamicAdvisedInterceptor)).getTargetSource().getTarget();
    currClass = target.getClass();
} else if (AopUtils.isJdkDynamicProxy(proxyClass)) {
    Field h = currClass.getSuperclass().getDeclaredField("h");
    h.setAccessible(true);
    AopProxy aopProxy = (AopProxy) h.get(proxyClass);
    Field advised = aopProxy.getClass().getDeclaredField("advised");
    advised.setAccessible(true);
    Object target = ((AdvisedSupport) advised.get(aopProxy)).getTargetSource().getTarget();
    currClass = target.getClass();
}
```

Spring主要通过AopProxy接口来实现Aop，主要包括JdkDynamicAopProxy和CglibAopProxy

getProxy方法返回代理对象。

* JdkDynamicAopProxy：JDK动态代理，通过JDK的InvocationHandler实现了AOP
* CglibAopProxy：通过CGLIB动态代理实现AOP

# SpringBoot

SpringBoot的启动代码：

```java
@SpringBootApplication
public class TestApp {
  public static void main(String[] args) {  
    SpringApplication.run(TestApp.class, args);  
  }
}

```

SpringBootApplication是EnableAutoConfiguration，SpringBootConfiguration，和ComponentScan的复合注解。

run方法会执行SpringApplication的run方法（非静态）：

```java
// Allows post-processing of the bean factory in context subclasses.
postProcessBeanFactory(beanFactory);

// 根据ComponentScan或ImportResource注解找到basePackage，
// classloader.getReources()找到目录，扫描目录的bean
// 执行自动装配
invokeBeanFactoryPostProcessors(beanFactory);

// 找到BeanPostProcessors，比如自定义注解处理器等
registerBeanPostProcessors(beanFactory);

// Initialize message source for this context.
initMessageSource();

// Initialize event multicaster for this context.
initApplicationEventMulticaster();

// 启动Tomcat
onRefresh();

// 检测定义的listeners
registerListeners();

// 初始化bean->注入bean->bean后处理->init方法
// 代理（AOP）在此被初始化，在注入bean的时候，判断是否为AOP proxy，通过下面的方法注入proxy
// 每个代理对象都会对应一个CglibAopProxy或者JdkDynamicAopProxy
//AbstractAutoProxyCreator.createProxy->ProxyFactory.getProxy(这里会根据配置决定创建CglibAopProxy或者JdkDynamicAopProxy)->CglibAopProxy.getProxy
finishBeanFactoryInitialization(beanFactory);

// 发布ContextRefreshedEvent事件
finishRefresh();

// 调用runner
afterRefresh(context, applicationArguments);
```

## Springboot事件监听

监听Springboot应用事件

```java
public static class ApplicationContextEventListener implements ApplicationListener<ApplicationContextEvent> {   
  public void onApplicationEvent(ApplicationContextEvent event) {    
  }
}
```

通过下面的方法注册到Spring中：

```java
SpringApplication application = new SpringApplication(XXXSpringBootApplication.class);
application.addListeners(new ApplicationContextEventListener());
```

事件类型：

* ContextRefreshedEvent(继承ApplicationContextEvent)(Springboot应用启动)： Published when the ApplicationContext is initialized or refreshed (for example, by using the refresh() method on the ConfigurableApplicationContext interface). Here, “initialized” means that all beans are loaded, post-processor beans are detected and activated, singletons are pre-instantiated, and the ApplicationContext object is ready for use. As long as the context has not been closed, a refresh can be triggered multiple times, provided that the chosen ApplicationContext actually supports such “hot” refreshes. For example, XmlWebApplicationContext supports hot refreshes, but GenericApplicationContext does not. 
* ContextStartedEvent： Published when the ApplicationContext is started by using the start() method on the ConfigurableApplicationContext interface. Here, “started” means that all Lifecycle beans receive an explicit start signal. Typically, this signal is used to restart beans after an explicit stop, but it may also be used to start components that have not been configured for autostart (for example, components that have not already started on initialization). 
* ContextStoppedEvent： Published when the ApplicationContext is stopped by using the stop() method on the ConfigurableApplicationContext interface. Here, “stopped” means that all Lifecycle beans receive an explicit stop signal. A stopped context may be restarted through a start() call. 
* ContextClosedEvent(Spring应用关闭)： Published when the ApplicationContext is closed by using the close() method on the ConfigurableApplicationContext interface. Here, “closed” means that all singleton beans are destroyed. A closed context reaches its end of life. It cannot be refreshed or restarted. 

**BeanPostProcessor**

BeanPostProcessor可以在Bean执行init方法或者destroy方法之前执行一些自定义的动作，比如给Bean的某些属性依据自定义的注解注入对象。

## Springboot自动装配原理

SpringBootApplication注解包含了ComponentScan注解，默认扫描与SpringBootApplication注解类同包下的bean。也可以自动装配其他jar包中指定的bean，按照约定大于配置的原则，其他jar包按照约定，将需要装配bean的配置放到spring.factories文件中，springboot就会自动装配这些bean。

springboot是通过EnableAutoConfiguration（被SpringBootApplication注解包含）这个注解进行自动装配的。

```java
@Import(EnableAutoConfigurationImportSelector.class->AutoConfigurationImportSelector.class->ImportSelector.class)
public @interface EnableAutoConfiguration {
```

ImportSelector接口配合Import注解使用，其中的`selectImports`接口根据配置返回需要加载的类。

AutoConfigurationImportSelector的selectImports方法会调用`SpringFactoriesLoader.loadFactoryNames`方法，遍历加载的jar包中META-INF中的spring.factories的`org.springframework.boot.autoconfigure.EnableAutoConfiguration`key值。将其对应的Configuration中配置的bean注入到容器中。

举例：

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration(注解）=org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

## BeanFactory和FactoryBean什么区别

* BeanFactory是Bean的工厂，用来管理Bean，提供诸如getBean的接口
* FactoryBean是一类特殊的Bean，通过BeanFactory.getBean接口获取的Bean，其实是FactoryBean的getObject接口返回的对象。

https://cloud.tencent.com/developer/article/1457284

**举例**

如果UserService这个Bean实现了FactoryBean这个接口，`BeanFactory.getBean("userService")`返回的对象是FactoryBean.getObject返回的值，如果FactoryBean配置的是单例模式，则每次调用getObject返回的对象是相同的，反之，则是不同的。如果就想获取到UserService这个Bean，则可以通过这种方式：`BeanFactory.getBean("&userService")`

FactoryBean的应用：AOP ProxyFactoryBean(手动实现AOP)
