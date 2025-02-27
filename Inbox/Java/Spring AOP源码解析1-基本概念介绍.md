---
tags:
    - Spring
    - SourceCode
---

## 关注问题

1. 什么是动态代理；
2. CGLIB 和 JDK 动态代理使用方式；
3. Spring 的 ProxyFactory 的用法和概念；
4. Spring 中代理的使用方式；
5. Advice 的用法和概念；
6. Advisor 的用法和概念；
7. Spring 中 AOP 的理解；

## 动态代理概念介绍

代理模式的解释：为其他对象提供一种代理以控制这个对象的访问，增强一个类中的某个方法，对程序进行扩展。

比如，存在一个类：
```java
public class UserService {  
    public void test() {  
        System.out.println("test...");  
    }  
}
```

此时，我们 `new` 一个 `UserService` 对象，然后执行 `test()` 方法，结果是显而易见的。

如果我们现在想在 **不修改 UserService 类的源码** 前提下，给 `test()` 方法增加额外逻辑，就可以使用动态代理机制来创建 `UserService`  对象了，比如：
```java
UserService target = new UserService();  
  
// 通过cglib技术  
Enhancer enhancer = new Enhancer();  
enhancer.setSuperclass(UserService.class);  
  
// 定义额外逻辑，也就是代理逻辑  
enhancer.setCallbacks(new Callback[]{new MethodInterceptor() {  
    @Override  
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {  
        System.out.println("before...");  
        Object result = methodProxy.invoke(target, objects);  
        System.out.println("after...");  
        return result;  
    }  
}});  
  
// 动态代理所创建出来的UserService对象  
UserService userService = (UserService) enhancer.create();  
userService.test();
```

这时候执行结果如下：
```java
before...
test...
after...
```
得到的都是 UserService 对象，但是执行 `test()`  方法时的效果却不一样了，这就是代理所带来的效果。

我们把上述代码中的 `Object result = methodProxy.invoke(target, objects)` 替换成 `Object result = methodProxy.invokeSuper(o, objects)` ，执行结果也是一样。

这里的 `target` 参数表示的是被代理的对象，就是目标对象 `UserService`，`invoke` 相当于执行目标对象的方法。那么 `invokeSuper` 怎么理解？
之前我们在文 Spring 底层核心原理简单介绍中 有大概介绍了 Spring 代理的基本原理，**invokeSuper 相当于调用目标对象的父类的方法。** 

### CGLIB 实现不同的方法不同的代理

比如，UserService 有两个方法：
```java
public class UserService {  
    public void test() {  
        System.out.println("test...");  
    }  
    public void a() {  
        System.out.println("UserService a...");  
    }  
}
```

我们想实现不同的方法调用不同的代理：
```java
UserService target = new UserService();  
  
Enhancer enhancer = new Enhancer();  
enhancer.setSuperclass(UserService.class);  
  
enhancer.setCallbacks(new Callback[]{new MethodInterceptor() {  
    @Override  
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {  
        System.out.println("before...");  
        Object result = methodProxy.invoke(target, objects);  
        System.out.println("after...");  
        return result;  
    }  
}, NoOp.INSTANCE});  
  
enhancer.setCallbackFilter(new CallbackFilter() {  
    @Override  
    public int accept(Method method) {  
        if (method.getName().equals("test")) {  
            return 0;  
        } else {  
            return 1;  
        }  
    }  
});  
  
UserService userService = (UserService) enhancer.create();  
userService.test();  
userService.a();
```

上述代码中，和原先不一样的是 `CallBack[]` 这个数组里面有两个代理，` NoOp.INSTANCE `  表示一个空的代理，啥也不执行。

`CallbackFilter` 是代理过滤器，返回的数字表示执行的是 `Callback` 数组中的哪个代理。


### JDK 动态代理

除了  CGLIB，JDK 本身也提供了一种创建代理对象的动态代理机制，但是它只能代理接口，也就是 UserService 得先有一个接口才能利用 JDK 动态代理机制来生成一个代理对象，比如：
```java
public interface UserInterface {
	public void test();
}
public class UserService implements UserInterface {
	public void test() {
		System.out.println("test...");
	}
}
```

```java
UserService target = new UserService();

// UserInterface接口的代理对象
Object proxy = Proxy.newProxyInstance(UserService.class.getClassLoader(), new Class[]{UserInterface.class}, new InvocationHandler() {
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		System.out.println("before...");
		Object result = method.invoke(target, args);
		System.out.println("after...");
		return result;
	}
});

UserInterface userService = (UserInterface) proxy;
userService.test();
```

如果把 `new Class[]{UserInterface.class}` ，替换成 `new Class[]{UserService.class}` ，代码会直接报错，由于这个限制，所以产生的代理对象的类型是 UserInterface，而不是 UserService，这是需要注意的。

## Spring 中的 ProxyFactory 

上面我们介绍了两种动态代理技术，那么在 Spring 中进行了封装，封装出来的类叫做 `ProxyFactory`，表示是创建代理对象的一个工厂，使用起来会比上面的更加方便，比如：
```java
UserService target = new UserService();

ProxyFactory proxyFactory = new ProxyFactory();
proxyFactory.setTarget(target);
proxyFactory.addAdvice(new MethodInterceptor() {
	@Override
	public Object invoke(MethodInvocation invocation) throws Throwable {
		System.out.println("before...");
		Object result = invocation.proceed();
		System.out.println("after...");
		return result;
	}
});

UserInterface userService = (UserInterface) proxyFactory.getProxy();
userService.test();
```

通过 `ProxyFactory`，我们可以不再关系到底是用 CGLIB 还是 JDK 动态代理了，`ProxyFactory` 会帮我们去判断。如果 `UserService` 实现了接口，那么 `ProxyFactory` 底层就会用 jdk 动态代理；如果没有实现接口，就会用 CGLIB 技术。
上面的代码，就是由于 `UserService` 实现了 `UserInterface` 接口，所以最后产生的代理对象是 `UserInterface` 类型。


### Advice 方法

上述代码中，我们 new 了一个 `Advice` 来执行，实际上我们可以定义多种 `Advice` 类型。

#### 方法之前执行 

```java
public class MyBeforeAdvice implements MethodBeforeAdvice {  
    @Override  
    public void before(Method method, Object[] args, Object target) throws Throwable {  
        System.out.println("方法执行前执行");  
    }  
}
```
其中，`method` 表示代理方法，`args` 表示传递给方法的参数，`target` 表示被代理对象。

#### 方法 return 后执行

```java
public class MyAfterReturnAdvice implements AfterReturningAdvice {  
    @Override  
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {  
        System.out.println("方法return后执行");  
    }  
}
```
`returnValue` 表示方法返回值。

#### 方法抛异常时执行

```java
public class MyThrowAdvice implements ThrowsAdvice {  
    public void afterThrowing(Method method, Object[] args, Object target, NullPointerException exception) {  
        System.out.println("方法抛出异常后执行");  
    }  
}
```
这里捕获的是空指针异常，我们也可以捕获其他异常，把 `NullPointerException`  换成其他异常即可。


#### 自定义执行顺序

这是功能最强大的 `Advice`，可以自定义执行顺序。例如：
```java
public class MyAroundAdvice implements MethodInterceptor {  
    @Nullable  
    @Override    
    public Object invoke(@Nonnull MethodInvocation invocation) throws Throwable {  
        System.out.println("方法执行Around前");  
        Object proceed = invocation.proceed();  
        System.out.println("方法执行Around后");  
        return proceed;  
    }  
}
```

#### 多个Advice

这些代理方法还可以组成一个链路，例如：
```java
UserService target = new UserService();  
  
ProxyFactory proxyFactory = new ProxyFactory();  
proxyFactory.setTarget(target);  
proxyFactory.addAdvice(new MyAroundAdvice());  
proxyFactory.addAdvice(new MyBeforeAdvice());  
proxyFactory.addAdvice(new MyAfterReturnAdvice());  
  
UserService proxy = (UserService) proxyFactory.getProxy();  
proxy.test();
```

执行结果：
```java
方法执行Around前
方法执行前执行
test...
方法return后执行
方法执行Around后
```

其中**执行的链路顺序是和方法  `addAdvice`  的添加顺序一致。**

### Advisor

跟 `Advice` 类似的还有一个 `Advisor` 的概念，一个 `Advisor` 是有一个 `Pointcut` 和一个 `Advice` 组成的，通过 `Pointcut` 可以指定要需要被代理的逻辑，比如一个 `UserService` 类中有两个方法，按上面的例子，这两个方法都会被代理，被增强，那么我们现在**可以通过 `Advisor`，来控制到具体代理哪一个方法**，比如：
```java
UserService target = new UserService();  
  
ProxyFactory proxyFactory = new ProxyFactory();  
proxyFactory.setTarget(target);  
proxyFactory.addAdvisor(new PointcutAdvisor() {  
    @Override  
    public Pointcut getPointcut() {  
        return new StaticMethodMatcherPointcut() {  
            @Override  
            public boolean matches(Method method, Class<?> targetClass) {  
                return method.getName().equals("test");  
            }  
        };  
    }  
    @Override  
    public Advice getAdvice() {  
        return new MyBeforeAdvice();  
    }  
    @Override  
    public boolean isPerInstance() {  
        return false;  
    }  
});  
  
UserService userService = (UserService) proxyFactory.getProxy();  
userService.test();  
userService.a();
```

执行结果：
```java
方法执行前执行
test...
UserService a...
```

其中 Pointcut  是用来定义 Advice 这段代理逻辑要执行在哪个方法上面。

这里，我们可以很方便的通过 ProxyFactory 来代理对象，但是还并没有和 Spring 中的 Bean 很方便的结合起来，接着我们就来介绍 Spring  中创建代理对象的方式。


### 创建代理对象的方式

上面介绍了 Spring 中所提供了 ProxyFactory、Advisor、Advice、PointCut 等技术来实现代理对象的创建，但是我们在使用 Spring 时，我们并不会直接这么去使用 ProxyFactory。
比如说，我们希望 ProxyFactory 所产生的代理对象能直接就是 Bean，能直接从 Spring 容器中得到 UserSerivce 的代理对象。而这些，Spring 都是支持的，只不过作为开发者的我们肯定得告诉 Spring，那些类需要被代理，代理逻辑是什么。

#### ProxyFactoryBean

```java
@Bean
public ProxyFactoryBean userServiceProxy(){
	UserService userService = new UserService();

	ProxyFactoryBean proxyFactoryBean = new ProxyFactoryBean();
	proxyFactoryBean.setTarget(userService);
	proxyFactoryBean.addAdvice(new MethodInterceptor() {
		@Override
		public Object invoke(MethodInvocation invocation) throws Throwable {
			System.out.println("before...");
			Object result = invocation.proceed();
			System.out.println("after...");
			return result;
		}
	});
	return proxyFactoryBean;
}

```

通过这种方法来定义一个 UserService 的 Bean，并且是经过了 AOP 的。但是这种方式**只能针对某一个 Bean**。它是一个 FactoryBean，所以利用的就是 FactoryBean 技术，间接的将 UserService 的代理对象作为了 Bean。

ProxyFactoryBean 还有额外的功能，比如可以把某个 Advice 或 Advisor 定义成为 Bean，然后在 ProxyFactoryBean 中进行设置：
```java
@Bean
public MethodInterceptor myAroundAdvice(){
	return new MethodInterceptor() {
		@Override
		public Object invoke(MethodInvocation invocation) throws Throwable {
			System.out.println("before...");
			Object result = invocation.proceed();
			System.out.println("after...");
			return result;
		}
	};
}

@Bean
public ProxyFactoryBean userService(){
	UserService userService = new UserService();
	
    ProxyFactoryBean proxyFactoryBean = new ProxyFactoryBean();
	proxyFactoryBean.setTarget(userService);
	proxyFactoryBean.setInterceptorNames("myAroundAdvice");
	return proxyFactoryBean;
}
```


#### BeanNameAutoProxyCreator

`ProxyFactoryBean` 得自己指定被代理的对象，那么我们可以通过 `BeanNameAutoProxyCreator` 来通过指定某个 Bean 的名字，来对该 Bean 进行代理：
```java
@Bean
public BeanNameAutoProxyCreator beanNameAutoProxyCreator() {
	BeanNameAutoProxyCreator beanNameAutoProxyCreator = new BeanNameAutoProxyCreator();
	beanNameAutoProxyCreator.setBeanNames("userSe*");
	beanNameAutoProxyCreator.setInterceptorNames("myAroundAdvice");
	beanNameAutoProxyCreator.setProxyTargetClass(true);

    return beanNameAutoProxyCreator;
}
```

通过 `BeanNameAutoProxyCreator` 可以对批量的 Bean 进行 AOP，并且指定了代理逻辑，指定了一个 `InterceptorName`，也就是一个 `Advice`，前提条件是这个 `Advice` 也得是一个 Bean，这样 Spring 才能找到的，但是 `BeanNameAutoProxyCreator` 的缺点很明显，它只能根据 beanName 来指定想要代理的 Bean。

#### DefaultAdvisorAutoProxyCreator

```java
@Bean
public DefaultPointcutAdvisor defaultPointcutAdvisor(){
	NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
	pointcut.addMethodName("test");

    DefaultPointcutAdvisor defaultPointcutAdvisor = new DefaultPointcutAdvisor();
	defaultPointcutAdvisor.setPointcut(pointcut);
	defaultPointcutAdvisor.setAdvice(new ZhouyuAfterReturningAdvice());

    return defaultPointcutAdvisor;
}

@Bean
public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
    DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
	return defaultAdvisorAutoProxyCreator;
}
```

通过 `DefaultAdvisorAutoProxyCreator` 会直接去找所有 Advisor 类型的 Bean，根据 Advisor 中的 PointCut 和 Advice 信息，确定要代理的 Bean 以及代理逻辑。

那么，有没有更简单的方法呢？有的，那就是注解。
我们只定义一个类，然后通过在类中的方法上通过某些注解，来定义 PointCut 以及 Advice 比如： 
```java
@Aspect
@Component
public class MyAspect {
	@Before("execution(public void com.vitahlin.service.UserService.test())")
	public void myBefore(JoinPoint joinPoint) {
		System.out.println("myBefore");
	}
}
```

通过上面这个类，我们就直接定义好了所要代理的方法(通过一个表达式)，以及代理逻辑（被@Before 修饰的方法），简单明了。
这样对于 Spring 来说，它要做的就是来解析这些注解了，解析之后得到对应的 Pointcut 对象、Advice 对象，生成 Advisor 对象，扔进 ProxyFactory 中，进而产生对应的代理对象，具体怎么解析这些注解就是 **@EnableAspectJAutoProxy**  注解所要做的事情了，后面详细分析。 


## Spring 中 AOP 的理解

OOP 表示面向对象编程，是一种编程思想，AOP 表示面向切面编程，也是一种编程思想，而我们上面所描述的就是 Spring 为了让程序员更加方便的做到面向切面编程所提供的技术支持，换句话说，就是 Spring 提供了一套机制，可以让我们更加容易的来进行 AOP，所以这套机制我们也可以称之为 Spring AOP。

但是值得注意的是，上面所提供的注解的方式来定义 Pointcut 和 Advice，Spring 并不是首创，首创是 AspectJ，而且也不仅仅只有 Spring 提供了一套机制来支持 AOP，还有比如 JBoss 4.0、aspectwerkz 等技术都提供了对于 AOP 的支持。

而刚刚说的注解的方式，Spring 是依赖了 AspectJ 的，或者说，Spring 是直接把 AspectJ 中所定义的那些注解直接拿过来用，自己没有再重复定义了，不过也仅仅只是把注解的定义赋值过来了，每个注解具体底层是怎么解析的，还是 Spring 自己做的，所以我们在用 Spring 时，如果你想用@Before、@Around 等注解，是需要单独引入 aspecj 相关 jar 包的：
```java
compile group: 'org.aspectj', name: 'aspectjrt', version: '1.9.5'
compile group: 'org.aspectj', name: 'aspectjweaver', version: '1.9.5'
```

值得注意的是：AspectJ 是在编译时对字节码进行了修改，是直接在 UserService 类对应的字节码中进行增强的，也就是可以理解为是在编译时就会去解析 `@Before`  这些注解，然后得到代理逻辑，加入到被代理的类中的字节码中去的，所以 **如果想用 AspectJ 技术来生成代理对象，是需要用单独的 AspectJ 编译器的** 。我们在项目中很少这么用，我们仅仅只是用了@Before 这些注解，而我们在启动 Spring 的过程中，Spring 会去解析这些注解，然后利用动态代理机制生成代理对象的。

IDEA 中使用 AspectJ 可以参考： https://blog.csdn.net/gavin_john/article/details/80156963

### AOP 中的概念

上面我们已经提到 Advisor、Advice、PointCut 等概念了，还有一些其他的概念，首先关于 AOP 中的概念本身是比较难理解的，Spring 官网上是这么说的：
> Let us begin by defining some central AOP concepts and terminology. These terms are not Spring-specific. Unfortunately, AOP terminology is not particularly intuitive. However, it would be even more confusing if Spring used its own terminology.

意思是，AOP 中的这些概念不是 Spring 特有的，不幸的是，AOP 中的概念不是特别直观的，但是，如果 Spring 重新定义自己的那可能会导致更加混乱。

1. Aspect：表示切面，比如被 `@Aspect`  注解的类就是切面，可以在切面中去定义 Pointcut、Advice 等等；
2. Join point：表示连接点，表示一个程序在执行过程中的一个点，比如一个方法的执行，比如一个异常的处理，在 Spring AOP 中，一个连接点通常表示一个方法的执行；
3. Advice：表示通知，表示在一个特定连接点上所采取的动作。Advice 分为不同的类型，后面详细讨论，在很多 AOP 框架中，包括 Spring，会用 Interceptor 拦截器来实现 Advice，并且在连接点周围维护一个 Interceptor 链；
4. Pointcut：表示切点，用来匹配一个或多个连接点，Advice 与切点表达式是关联在一起的，Advice 将会执行在和切点表达式所匹配的连接点上；
5. Introduction：可以使用 @DeclareParents 来给所匹配的类添加一个接口，并指定一个默认实现；
6. Target object：目标对象，被代理对象；
7. AOP proxy：表示代理工厂，用来创建代理对象的，在 Spring Framework 中，要么是 JDK 动态代理，要么是 CGLIB 代理；
8. Weaving：表示织入，表示创建代理对象的动作，这个动作可以发生在编译时期（比如 Aspejctj），或者运行时，比如 Spring AOP。

### Advice 在 Spring AOP 中对应 API 

上面说到的 Aspject 中的注解，其中有五个是用来定义 Advice 的，表示代理逻辑，以及执行时机：
1. @Before
2. @AfterReturning
3. @AfterThrowing
4. @After
5. @Around

Spring 自己也提供了类似的执行实际的实现类：
1. 接口 MethodBeforeAdvice，继承了接口 BeforeAdvice
2. 接口 AfterReturningAdvice
3. 接口 ThrowsAdvice
4. 接口 AfterAdvice
5. 接口 MethodInterceptor

Spring 会把五个注解解析为对应的 Advice 类：
1. @Before ：AspectJMethodBeforeAdvice，实际上就是一个 MethodBeforeAdvice；
2. @AfterReturning：AspectJAfterReturningAdvice，实际上就是一个 AfterReturningAdvice；
3. @AfterThrowing：AspectJAfterThrowingAdvice，实际上就是一个 MethodInterceptor；
4. @After：AspectJAfterAdvice，实际上就是一个 MethodInterceptor；
5. @Around：AspectJAroundAdvice，实际上就是一个 MethodInterceptor。

### TargetSource 的使用

在我们日常的 AOP 中，被代理对象就是 Bean 对象，是由 `BeanFactory` 给我们创建出来的，但是 Spring AOP 中提供了 TargetSource 机制，可以让我们用来自定义逻辑来创建 **被代理对象**。 

比如之前所提到的 `@Lazy`  注解，当加在属性上时，会产生一个代理对象赋值给这个属性，产生代理对象的代码为：
```java
protected Object buildLazyResolutionProxy(final DependencyDescriptor descriptor, final @Nullable String beanName) {  
      BeanFactory beanFactory = getBeanFactory();  
      Assert.state(beanFactory instanceof DefaultListableBeanFactory,  
            "BeanFactory needs to be a DefaultListableBeanFactory");  
      final DefaultListableBeanFactory dlbf = (DefaultListableBeanFactory) beanFactory;  
  
      TargetSource ts = new TargetSource() {  
         @Override  
         public Class<?> getTargetClass() {  
            return descriptor.getDependencyType();  
         }  
         @Override  
         public boolean isStatic() {  
            return false;  
         }  
         @Override  
         public Object getTarget() {  
            Set<String> autowiredBeanNames = (beanName != null ? new LinkedHashSet<>(1) : null);  
            Object target = dlbf.doResolveDependency(descriptor, beanName, autowiredBeanNames, null);  
            if (target == null) {  
               Class<?> type = getTargetClass();  
               if (Map.class == type) {  
                  return Collections.emptyMap();  
               }  
               else if (List.class == type) {  
                  return Collections.emptyList();  
               }  
               else if (Set.class == type || Collection.class == type) {  
                  return Collections.emptySet();  
               }  
               throw new NoSuchBeanDefinitionException(descriptor.getResolvableType(),  
                     "Optional dependency not present for lazy injection point");  
            }  
            if (autowiredBeanNames != null) {  
               for (String autowiredBeanName : autowiredBeanNames) {  
                  if (dlbf.containsBean(autowiredBeanName)) {  
                     dlbf.registerDependentBean(autowiredBeanName, beanName);  
                  }  
               }  
            }  
            return target;  
         }  
         @Override  
         public void releaseTarget(Object target) {  
         }      };  
  
      ProxyFactory pf = new ProxyFactory();  
      pf.setTargetSource(ts);  
      Class<?> dependencyType = descriptor.getDependencyType();  
      if (dependencyType.isInterface()) {  
         pf.addInterface(dependencyType);  
      }  
      return pf.getProxy(dlbf.getBeanClassLoader());  
   }  
}
```

其实上文中 `proxyFactory.setTarget(target)` 中的 setTarget 方法实际上也是设置了 TargetSource，我们可以看 setTarget 方法的源码 `AdvisedSupport#setTarget` ：
```java
public void setTarget(Object target) {  
   setTargetSource(new SingletonTargetSource(target));  
}
```

介绍完 AOP 的概念和 Spring 中 ProxyFactory 的用法，为了更透彻的了解 ProxyFactory，接下来我们就从源码的角度来分析 Spring 的 AOP 原理，也就是 ProxyFactory 的底层原理。

