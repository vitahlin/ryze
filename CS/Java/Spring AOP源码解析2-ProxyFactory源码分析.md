---
tags:
  - Spring
  - SourceCode
---


## 关注问题

1. ProxyFactory 如何选择 JDK 或 CGLIB；
2. MethodInterceptor 适配流程；
3. MethodInterceptor 列表执行流程；

## 前言

本文主要介绍 Spring ProxyFactory 的源码，在这之前需要对 Spring AOP 的概念有所了解，参考[[Spring AOP源码解析1-基本概念介绍]]

## ProxyFactory 的简单使用

先来看一下 `ProxyFactory` 的简单使用：
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

UserService userService = (UserService)proxyFactory.getProxy();
userService.test();
```

上文提到的 ProxyFactory 例子中，先创建了一个 ProxyFactory 实例，然后设置了一些参数，最后调用 `ProxyFactory#getProxy` 方法创建了一个对象，这个对象就是增强后的代理对象，调用增强后的实例的 hello 方法，可以发现输出内容已经被增强。

为了了解其实现原理，可以直接看 `getProxy` 方法的源码。
```java
public Object getProxy() {  
   return createAopProxy().getProxy();  
}
```

该方法的源码很简单，分为两个部分，分别是 `createAopProxy` 和 `getProxy`。
从方法名可以看出，`createAopProxy` 是用来创建一个 `AopProxy`，`getProxy` 是获取代理对象。

## AopProxy

我们先看一下 `createAopProxy` 方法返回的 `AopProxy` 是个什么：
```java
public interface AopProxy {
	Object getProxy();
	Object getProxy(@Nullable ClassLoader classLoader);
}
```

`AopProxy` 是一个接口，只有两个方法。通过方法名就可以猜到，这两个方法都是用来获取代理类，一个是不需要指定 `ClassLoader`，另外一个可以指定 `ClassLoader`，说明这两个方法的本质是一样的。

再看一下这个类的注释：
> Delegate interface for a configured AOP proxy, allowing for the creation of actual proxy objects.  
> Out-of-the-box implementations are available for JDK dynamic proxies and for CGLIB proxies, as applied by DefaultAopProxyFactory.

通过注释和定义的方法，可以大概了解到，这个接口的实现类是用来创建代理对象的，而其实现类可以通过 JDK 动态代理或者 CGLIB 来创建代理。

## createAopProxy-选择 JDK 还是 CGLIB 来创建动态代理

对 `AopProxy` 有大致了解以后，再回到之前的代码中，看一下 `createAopProxy` 方法的代码，可以发现 `createAopProxy` 是在 `ProxyFactory` 的父类 `ProxyCreatorSupport` 中定义的。 `ProxyCreatorSupport.createAopProxy` :
```java
private AopProxyFactory aopProxyFactory;

public ProxyCreatorSupport() {  
this.aopProxyFactory = new DefaultAopProxyFactory();  
}

public AopProxyFactory getAopProxyFactory() {  
   return this.aopProxyFactory;  
}

protected final synchronized AopProxy createAopProxy() {  
   if (!this.active) {  
      activate();  
   }  
   return getAopProxyFactory().createAopProxy(this);  
}
```

主要关注方法 `getAopProxyFactory` 和 `createAopProxy`。还是通过方法名先猜一下这两个方法的含义，`getAopProxyFactory` 是获取一个 `AopProxy` 工厂，`createAopProxy` 是通过工厂创建 `AopProxy` 实例。这样的逻辑也符合当前方法需要创建 `AopProxy` 的目的。

先来看方法 `getAopProxyFactory`  的实现。这个方法直接返回了当前实例持有的 ` AopProxyFactory ` 对象，那么这个对象在哪里创建的呢？就在当前类的构造方法中实例化。在构造方法中，创建了一个 `DefaultAopProxyFactory` 对象，这个类在上文将 `AopProxy` 的时候提到过，`DefaultAopProxyFactory`  是 ` AopProxyFactory ` 接口的唯一实现。

跟进 `DefaultAopProxyFactory`  的方法里面，Spring 在这里面就会判断到底是用 CGLIB 还是 JDK 实现动态代理： 
```java
@Override  
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {  
    if (!NativeDetector.inNativeImage() &&  
        (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {  
        Class<?> targetClass = config.getTargetClass();  
        if (targetClass == null) {  
            throw new AopConfigException("TargetSource cannot determine target class: " +  
                "Either an interface or a target is required for proxy creation.");  
        }  
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {  
            return new JdkDynamicAopProxy(config);  
        }  
        return new ObjenesisCglibAopProxy(config);  
    } else {  
        return new JdkDynamicAopProxy(config);  
    }  
}
```

`NativeDetector.inNativeImage` 这个会判断当前 Spring 运行的 JVM，如果是运行在 GraalVM，那就直接用 JDK 动态代理，如果不是则继续判断其他条件。

接下来是判断 config 某些属性，那么这个 config  是什么？在 createAopProxy 方法中可以看到，传入的参数是 this，其实就是我们自己 new 出来的 ProxyFactory，ProxyFactory 这个类继承了 ProxyConfig 类。

我们的 ProxyFactory 可以设置一些属性，比如 optimize。在以前的版本中，大部分情况下 CGLIB 的效率会比 JDK 的高一点，如果想提高效率，就可以设置 `optimize=true`，这样它就会选择 CGLIB 来进行动态代理。

`config.isProxyTargetClass` 判断被代理的是不是类，**因为 JDK 只能代理接口**。Spring 的注解 `@EnableAspectJAutoProxy` 中的 proxyTargetClass 其实就是设置这个属性，它的默认值是 false，如果设为 true，那么就会直接使用 CGLIB，不关心代理类是否实现了接口。

如果前面两者都是 false，那么就判断 `hasNoUserSuppliedProxyInterfaces` ，这个方法是判断 proxyFactory  有没有调用 addInterface 。
如果有就返回 false 使用 JDK，如果没有添加接口返回 true 就使用 CGLIB。这里的有没有添加接口并不是说 UserService 是否有实现接口，这里判断的是是否有调用 `proxyFactory.addInterface()`。

这段代码就是 Spring 判断需要通过什么方式来创建代理对象，在这里需要对 JDK 动态代理和 CGLIB 有一定的了解，例如动态代理要求必须有接口才可以进行代理，是通过实现父接口的方式创建代理的；CGLIB 是通过生成目标类型的子类进行代理的，那么这个目标类型必须不能是 final 的（final 类不能继承），而且已经 CGLIB 代理过一次的类型不能再次进行代理（因为 CGLIB 在代理时会生成一些特殊的方法，如果重复代理会导致方法冲突）。

到此为止，我们已经分析完了 `ProxyCreatorSupport#createAopProxy` 方法，已经得到了适合当前目标类型的 AopProxy。接下来便是分析 `AopProxy#getProxy`，由于 AopProxy 有两个实现类，分别是 JdkDynamicAopProxy 和 ObjenesisCglibAopProxy，我们分别看一下这两个类的 getProxy 代码。


## JdkDynamicAopProxy -JDK 动态代理

`JdkDynamicAopProxy` 的 getProxy 方法比较简单，调用了同名的重载方法，传入了当前的默认 `ClassLoader`：
```java
public Object getProxy() {
    return getProxy(ClassUtils.getDefaultClassLoader());
}
```

继续来看带参数的 `JdkDynamicAopProxy#getProxy` 方法：
```Java  
    @Override
    public Object getProxy(@Nullable ClassLoader classLoader) {
        if (logger.isTraceEnabled()) {
            logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
        }
        /*** 这里InvocationHandler传入的参数是this,最终执行的当前类invoke方法 */
        return Proxy.newProxyInstance(classLoader, this.proxiedInterfaces, this);
    }
```

可以看到，这里需要传入 `InvocationHandler` 对象地方传入了 `this`，当前类 `JdkDynamicAopProxy` 也实现了 `InvocationHandler` 接口，那么看一下当前类的 `invoke` 方法。

### JdkDynamicAopProxy#invoke

直接来看 `JdkDynamicAopProxy#invoke`  源码：
```Java
    @Override
    @Nullable
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object oldProxy = null;
        boolean setProxyContext = false;

        /**
         * 拿到被代理对象。
         * 这里的this.advised就是我们的ProxyFactory对象
         */
        TargetSource targetSource = this.advised.targetSource;
        Object target = null;

        try {
            /***
             * 如果被代理的目标对象要执行的方法是equal,
             * 则执行JdkDynamicAopProxy（即代理对象的equal）方法，
             * 然后就返回了，也就说spring不对equal方法进行AOP拦截.
             * 接下来的hashCode方法类似
             */
            if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
                // The target does not implement the equals(Object) method itself.
                return equals(args[0]);
            } else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
                // The target does not implement the hashCode() method itself.
                return hashCode();
            } else if (method.getDeclaringClass() == DecoratingProxy.class) {
                // There is only getDecoratedClass() declared -> dispatch to proxy config.
                return AopProxyUtils.ultimateTargetClass(this.advised);
            } else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
                method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                // Service invocations on ProxyConfig with the proxy config...
                return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
            }

            Object retVal;

            /**
             * ProxyFactory中的exposeProxy属性为true，则将代理对象设置到currencyProxy这个ThreadLocal中去.
             * 设置为true才可以在我们自己写的方法里面拿到代理对象
             */
            if (this.advised.exposeProxy) {
                // Make invocation available if necessary.
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
            }

            // Get as late as possible to minimize the time we "own" the target,
            // in case it comes from a pool.

            /*** 被代理对象和代理类。*/
            target = targetSource.getTarget();
            Class<?> targetClass = (target != null ? target.getClass() : null);

            // Get the interception chain for this method.
            /**
             * 代理对象在执行某个方法时，根据方法筛选出匹配的Advisor，
             * 并适配成Interceptor。
             * method当前方法，targetClass当前被代理对象，这就是筛选符合这个方法的advisor.
             * 获取定义好的拦截器链，即Advice列表
             */
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

            // Check whether we have any advice. If we don't, we can fallback on direct
            // reflective invocation of the target, and avoid creating a MethodInvocation.
            if (chain.isEmpty()) {
                // We can skip creating a MethodInvocation: just invoke the target directly
                // Note that the final invoker must be an InvokerInterceptor so we know it does
                // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
                /*** 匹配的是空的，直接执行被代理对象target的method方法 */
                Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
            } else {
                // We need to create a method invocation...
                /**
                 * 存在代理的情况
                 * proxy 代理对象
                 * target 被代理对象
                 * method 执行的方法
                 * args 参数
                 * targetClass 被代理的类
                 * chain advise的链
                 */
                MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                // Proceed to the joinpoint through the interceptor chain.

                /*** proceed就是执行方法 */
                retVal = invocation.proceed();
            }

            // Massage return value if necessary.
            Class<?> returnType = method.getReturnType();
            if (retVal != null && retVal == target &&
                returnType != Object.class && returnType.isInstance(proxy) &&
                !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                // Special case: it returned "this" and the return type of the method
                // is type-compatible. Note that we can't help if the target sets
                // a reference to itself in another returned object.
                retVal = proxy;
            } else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
                throw new AopInvocationException(
                    "Null return value from advice does not match primitive return type for: " + method);
            }
            return retVal;
        } finally {
            if (target != null && !targetSource.isStatic()) {
                // Must have come from TargetSource.
                targetSource.releaseTarget(target);
            }
            if (setProxyContext) {
                // Restore old proxy.
                AopContext.setCurrentProxy(oldProxy);
            }
        }
    }
```

该方法被回调时传入的三个参数分别是：被调用的代理对象、被调用的方法、调用参数列表。整体流程还是比较简单的，参考上述的注释即可，这里就不细讲了。这里主要关注 2 个方法，分别是
1.  `AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice` ，我们可能会定义多个 Advisor 或者 Advice，这个方法就是进行筛选符合这个类的 Advice 列表；如果没有符合的 Advice 列表，就调用 `invokeJoinpointUsingReflection` 执行普通的方法。
2. `ReflectiveMethodInvocation#proceed`


### getInterceptorsAndDynamicInterceptionAdvice-获取 Interceptor 列表

在 AOP 基本概念介绍中，我们可以定义多种 Advice，也有一个自定义执行顺序的 Advice，如下：
```Java
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

可以看到它是通过 `MethodInterceptor` 实现的，而且它是最为灵活的，可以随意控制它的执行顺序。而其他的 Advice，其实是 Spring 为了更加方便我们使用所设计的一些接口，底层实际上还是通过 `MethodInterceptor` 所实现的。

这时候我们再来这个方法名就很容易理解了，get Interceptors 就是获取一个 `MethodInterceptor` 的列表，直接来看方法源码 `AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice` :
```Java
    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
        MethodCacheKey cacheKey = new MethodCacheKey(method);

        /*** cached就是表示advice链 */
        List<Object> cached = this.methodCache.get(cacheKey);
        if (cached == null) {
            /**
             * 我们定义的时候除了Around是定义MethodInterceptor，
             * 其他都有现成的Advice接口，但是实际上Advice也会转换成MethodInterceptor
             */
            cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(this, method, targetClass);
            this.methodCache.put(cacheKey, cached);
        }
        return cached;
    }
```

这个方法进行了缓存处理，而实际执行的是其实现类 `DefaultAdvisorChainFactory` ，继续跟进代码 `DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice` ：
```Java
 @Override
    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
        Advised config, Method method, @Nullable Class<?> targetClass) {

        // This is somewhat tricky... We have to process introductions first,
        // but we need to preserve order in the ultimate list.
        AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

        /**
         * config就是我们上文定义的ProxyFactory.
         * 这里就是获取所有的Advisor，proxyFactory.addAdvice调用的时候其实就是添加Advisor
         */
        Advisor[] advisors = config.getAdvisors();
        List<Object> interceptorList = new ArrayList<>(advisors.length);
        Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
        Boolean hasIntroductions = null;

        /*** 遍历所有的advisor */
        for (Advisor advisor : advisors) {
            if (advisor instanceof PointcutAdvisor) {
                // Add it conditionally.
                PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
                /*** 判断被代理的类是不是匹配,即先筛选一遍ClassFilter */
                if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                    /*** 类匹配，就获取匹配的方法 */
                    MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                    boolean match;
                    if (mm instanceof IntroductionAwareMethodMatcher) {
                        if (hasIntroductions == null) {
                            hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
                        }
                        match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
                    } else {
                        /*** 判断方法是不是匹配 */
                        match = mm.matches(method, actualClass);
                    }


                    if (match) {
                        /*** 发现匹配，现在传入的是一个advisor，转换为MethodInterceptor */
                        MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                        if (mm.isRuntime()) {
                            // Creating a new object instance in the getInterceptors() method
                            // isn't a problem as we normally cache created chains.
                            for (MethodInterceptor interceptor : interceptors) {
                                interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                            }
                        } else {
                            interceptorList.addAll(Arrays.asList(interceptors));
                        }
                    }
                }
            } else if (advisor instanceof IntroductionAdvisor) {
                IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
                if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                    Interceptor[] interceptors = registry.getInterceptors(advisor);
                    interceptorList.addAll(Arrays.asList(interceptors));
                }
            } else {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }

        return interceptorList;
    }
```

#### 获取全部 Advisor

首先会先执行 `config.getAdvisors()` 方法获取 Advisor 列表，那么这里为什么是 Advisor 而不是 Advice 呢？一开始的示例代码中分明执行的是 addAdvice 方法：
```Java
proxyFactory.addAdvice(new MethodInterceptor() {
	@Override
	public Object invoke(MethodInvocation invocation) throws Throwable {
		System.out.println("before...");
		Object result = invocation.proceed();
		System.out.println("after...");
		return result;
	}
});
```

实际上，proxyFactory 在调用 `addAdvice`  方法的时候会进行转化，我们直接来看 `addAdvice`  方法的源码 ` AdvisedSupport#addAdvice` ：
```Java
    @Override
    public void addAdvice(Advice advice) throws AopConfigException {
        int pos = this.advisors.size();
        addAdvice(pos, advice);
    }
```

```Java
  @Override
    public void addAdvice(int pos, Advice advice) throws AopConfigException {
        Assert.notNull(advice, "Advice must not be null");
        if (advice instanceof IntroductionInfo) {
            // We don't need an IntroductionAdvisor for this kind of introduction:
            // It's fully self-describing.
            addAdvisor(pos, new DefaultIntroductionAdvisor(advice, (IntroductionInfo) advice));
        } else if (advice instanceof DynamicIntroductionAdvice) {
            // We need an IntroductionAdvisor for this kind of introduction.
            throw new AopConfigException("DynamicIntroductionAdvice may only be added as part of IntroductionAdvisor");
        } else {
            addAdvisor(pos, new DefaultPointcutAdvisor(advice));
        }
    }
```
会发现，最终调用的是 `addAdvisor(pos, new DefaultPointcutAdvisor(advice))` ，advice 会转化为 `DefaultPointcutAdvisor` 。

#### 遍历 Advisor 列表

##### ClassFilter 和 MethodMatcher 的筛选

依次会判断 `Advisor`  的 `ClassFilter` 和 `MethodMatcher`  是否符合条件，实际上一个完整体的 `Advisor`  长这样：
```Java
public class MyPointcutAdvisor implements PointcutAdvisor {
    @Override
    public Advice getAdvice() {
        return new MyBeforeAdvice();
    }

    @Override
    public boolean isPerInstance() {
        return false;
    }

    @Override
    public Pointcut getPointcut() {
        return new Pointcut() {
            @Override
            public ClassFilter getClassFilter() {
                return new ClassFilter() {
                    @Override
                    public boolean matches(Class<?> clazz) {
                        /*** 这里的参数是被代理的类，我们就可以自定义匹配的逻辑 */
                        return false;
                    }
                };
            }

            @Override
            public MethodMatcher getMethodMatcher() {
                return new MethodMatcher() {
                    @Override
                    public boolean matches(Method method, Class<?> targetClass) {
                        return false;
                    }

                    @Override
                    public boolean isRuntime() {
                        return false;
                    }

                    @Override
                    public boolean matches(Method method, Class<?> targetClass, Object... args) {
                        return false;
                    }
                };
            }
        };
    }
}

```

我们可以在 `ClassFilter` 和 `MethodMatcher` 定义筛选的逻辑，Spring 筛选 `Advisor`  的时候就会判断当前这个方法符合不符合筛选的逻辑。

上述中还有一个 `isRuntime`  的判断，这个是判断当前是不是运行时，即不止判断 ` ClassFilter `  和 ` MethodMatcher ` ，如果是运行时还会判断 ` matches(Method method, Class<?> targetClass, Object... args)` 这个方法来进行当前运行参数的筛选。


##### 将 Advisor 转换为 MethodInterceptor

匹配后，就会将 `Advisor` 转换为 `MethodInterceptor`：
```Java
MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
```


在看具体代码之前，先看一下 `AdvisorAdapterRegistry` 是用来干什么的，源码文档中对其的描述只有一句话：
> Interface for registries of Advisor adapters.

这是一个进行适配的工具，这里就用到了设计模式中的**适配器模式**。
既然是适配器模式，我们需要了解将什么适配成什么。其实能猜到，这个方法的入参是 `Advisor` ，而返回值是 `Interceptor` ，那么大概率就是这两个类型之间进行适配了。

这里虽然返回的是 `MethodInterceptor[]`，但是实际上一个 `Advisor` 只会转化为一个 `MethodInterceptor`，继续来看 `registry.getInterceptors` 方法的源码，`DefaultAdvisorAdapterRegistry#getInterceptors` ：
```Java
 @Override
    public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
        List<MethodInterceptor> interceptors = new ArrayList<>(3);
        
        /*** 取出advisor里面的advice */
        Advice advice = advisor.getAdvice();
        
        /*** 当前advice是MethodInterceptor直接加入List中 */
        if (advice instanceof MethodInterceptor) {
            interceptors.add((MethodInterceptor) advice);
        }
        /*** advice适配成MethodInterceptor */
        for (AdvisorAdapter adapter : this.adapters) {
            /*** supportsAdvice方法判断支持的类型 */
            if (adapter.supportsAdvice(advice)) {
                interceptors.add(adapter.getInterceptor(advisor));
            }
        }
        if (interceptors.isEmpty()) {
            throw new UnknownAdviceTypeException(advisor.getAdvice());
        }
        return interceptors.toArray(new MethodInterceptor[0]);
    }
```



先来看 AdvisorAdapter 接口的源码：
```Java
public interface AdvisorAdapter {
	boolean supportsAdvice(Advice advice);
	MethodInterceptor getInterceptor(Advisor advisor);
}
```

`AdvisorAdapter` 有两个方法，`supportsAdvice` 用来返回是否支持当前 `Advice`，`getInterceptor` 用来将支持的 `Advice` 适配成 `MethodInterceptor`。

AdvisorAdapter 接口的实现类有 4 个：
1. AfterReturningAdviceAdapter
2. MethodBeforeAdviceAdapter
3. SimpleBeforeAdviceAdapter
4. ThrowsAdviceAdapter

这里的逻辑就比较简单了，如果是 `MethodInterceptor` 就直接加入到列表中，否则就进行适配，适配的逻辑其实也比较简单，就是匹配是否是 Spring 定义的几种方便我们使用的 Advice，匹配的上也转化为 `MethodInterceptor`。

在这里 `this.adapters` 的值如下：
```Java
    public DefaultAdvisorAdapterRegistry() {
        registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
        registerAdvisorAdapter(new AfterReturningAdviceAdapter());
        registerAdvisorAdapter(new ThrowsAdviceAdapter());
    }
```


##### MethodBeforeAdviceAdapter 源码

随便看一个适配器的代码，比如 `MethodBeforeAdviceAdapter`，**这个很重要，下面会考**：
```Java
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

    @Override
    public boolean supportsAdvice(Advice advice) {
        return (advice instanceof MethodBeforeAdvice);
    }

    @Override
    public MethodInterceptor getInterceptor(Advisor advisor) {
        MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
        return new MethodBeforeAdviceInterceptor(advice);
    }
}
```

`supportsAdvice` 方法就是判断类，`getInterceptor` 方法就是拿到最终的 `MethodInterceptor`。

我们可以再顺便来看 `MethodBeforeAdviceInterceptor` 的源码，看代理逻辑是怎么执行的：
```Java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {

    private final MethodBeforeAdvice advice;

    public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
        Assert.notNull(advice, "Advice must not be null");
        this.advice = advice;
    }

    @Override
    @Nullable
    public Object invoke(MethodInvocation mi) throws Throwable {
        this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
        return mi.proceed();
    }
}
```

**关注其中的 invoke 方法，`this.advice.before` 就会先执行 `MethodBeforeAdvice`  的 `before`  方法，` mi.proceed ` 再执行代理的方法。**

所以，这里不管是 Advice 还是 Advisor，最终都会被处理成对应的 `MethodInterceptor` 列表。

### 执行 MethodInterceptor 列表

上述中，我们已经得到了当前的 MethodInterceptor 列表，那接下来肯定就是执行了。继续来看  `JdkDynamicAopProxy #invoke` 中的执行逻辑：
```Java
MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);  
/*** proceed就是执行方法 */  
retVal = invocation.proceed();
```

一开始会调用 `ReflectiveMethodInvocation`  的构造函数，继续看 proceed 的具体逻辑 ` ReflectiveMethodInvocation#proceed `：
```Java
 @Nullable
    public Object proceed() throws Throwable {
        // We start with an index of -1 and increment early.
        /*** this.currentInterceptorIndex初始值为-1 */
        if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
            return invokeJoinpoint();
        }

        /*** 取出下一个MethodInterceptor */
        Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
        if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
            // Evaluate dynamic method matcher here: static part will already have
            // been evaluated and found to match.
            /*** 如果是InterceptorAndDynamicMethodMatcher类型，这里还做方法参数的筛选 */
            InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
            Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
            if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
                return dm.interceptor.invoke(this);
            } else {
                // Dynamic matching failed.
                // Skip this interceptor and invoke the next in the chain.
                return proceed();
            }
        } else {
            // It's an interceptor, so we just invoke it: The pointcut will have
            // been evaluated statically before this object was constructed.
            /**
             * 调用MethodInterceptor的invoke方法，
             * 这里其实调用的就是各种MethodInterceptor实现类中的invoke方法
             */
            return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
        }
    }
```

这里主要思考的点在于，**我们要执行的是 MethodInterceptor 列表，为什么没有用 for 循环来处理**？

首先，`this.currentInterceptorIndex` 这个值一开始是 `-1`，

```Java
Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
```

就先取出列表的第一个 MethodInterceptor，然后调用当前 MethodInterceptor 的 invoke 方法，注意这里的 invoke 方法的参数是 this，然后回想我们刚才分析的 MethodBeforeAdviceAdapter 源码，它还是会调用 `mi.proceed()` ，又继续执行了这个流程，这时候 `this.currentInterceptorIndex` 的值加 1，直到全部的 Interceptor 执行完，这时候需要执行被代理的逻辑，即：
```Java
if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
    return invokeJoinpoint();
}
```


这就是整个 JDK 执行动态代理的流程。

## CglibAopProxy-CGLIB

在设置代理对象时，通过传入 `Callback`，当需要执行代理对象的所有方法时，将会回调传入的 `Callback` 对象，一般使用 `Callback` 的子接口 `org.springframework.cglib.proxy.MethodInterceptor`，实现其 `intercept` 方法，可以拦截目标方法进行增强。

接下来我们可以来看 CGLIB 的实现类，`CglibAopProxy#getProxy`：
```Java
 @Override
    public Object getProxy(@Nullable ClassLoader classLoader) {
        if (logger.isTraceEnabled()) {
            logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
        }
        
        try {
            /*** 被代理的类 */
            Class<?> rootClass = this.advised.getTargetClass();
            Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");
            
            Class<?> proxySuperClass = rootClass;
            /*** 如果被代理的类本身就是CGLIB所产生的类 */
            if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
                /*** 获取真正的代理类 */
                proxySuperClass = rootClass.getSuperclass();
                
                /*** 获取被代理类所实现的接口 */
                Class<?>[] additionalInterfaces = rootClass.getInterfaces();
                for (Class<?> additionalInterface : additionalInterfaces) {
                    this.advised.addInterface(additionalInterface);
                }
            }
            
            // Validate the class, writing log messages as necessary.
            validateClassIfNecessary(proxySuperClass, classLoader);
            
            /*** 这里就是CGLIB的Enhancer */
            Enhancer enhancer = createEnhancer();
            if (classLoader != null) {
                enhancer.setClassLoader(classLoader);
                if (classLoader instanceof SmartClassLoader &&
                    ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
                    enhancer.setUseCache(false);
                }
            }
            /*** 被代理类，代理类的父类 */
            enhancer.setSuperclass(proxySuperClass);
            /*** 代理类额外要实现的接口 */
            enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
            enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
            enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));
            
            /*** 获取和被代理类所匹配的Advisor */
            Callback[] callbacks = getCallbacks(rootClass);
            Class<?>[] types = new Class<?>[callbacks.length];
            for (int x = 0; x < types.length; x++) {
                types[x] = callbacks[x].getClass();
            }
            // fixedInterceptorMap only populated at this point, after getCallbacks call above
            enhancer.setCallbackFilter(new ProxyCallbackFilter(
                this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
            enhancer.setCallbackTypes(types);
            
            // Generate the proxy class and create a proxy instance.
            return createProxyClassAndInstance(enhancer, callbacks);
        } catch (CodeGenerationException | IllegalArgumentException ex) {
            throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
                ": Common causes of this problem include using a final class or a non-visible class",
                ex);
        } catch (Throwable ex) {
            // TargetSource.getTarget() failed
            throw new AopConfigException("Unexpected AOP exception", ex);
        }
    }
```

其实原理和 JDK 的差不多，获取 Callback 列表，然后筛选执行，

具体来看 getCallbacks 方法 `CglibAopProxy#getCallbacks`：
```Java
 private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
        // Parameters used for optimization choices...
        boolean exposeProxy = this.advised.isExposeProxy();
        boolean isFrozen = this.advised.isFrozen();
        boolean isStatic = this.advised.getTargetSource().isStatic();

        // Choose an "aop" interceptor (used for AOP calls).
        //
        Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

        // Choose a "straight to target" interceptor. (used for calls that are
        // unadvised but can return this). May be required to expose the proxy.
        Callback targetInterceptor;
        if (exposeProxy) {
            targetInterceptor = (isStatic ?
                new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
                new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource()));
        } else {
            targetInterceptor = (isStatic ?
                new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
                new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
        }

        // Choose a "direct to target" dispatcher (used for
        // unadvised calls to static targets that cannot return this).
        Callback targetDispatcher = (isStatic ?
            new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());

        Callback[] mainCallbacks = new Callback[]{
            aopInterceptor,  // for normal advice
            targetInterceptor,  // invoke target without considering advice, if optimized
            new SerializableNoOp(),  // no override for methods mapped to this
            targetDispatcher, this.advisedDispatcher,
            new EqualsInterceptor(this.advised),
            new HashCodeInterceptor(this.advised)
        };

        Callback[] callbacks;

        // If the target is a static one and the advice chain is frozen,
        // then we can make some optimizations by sending the AOP calls
        // direct to the target using the fixed chain for that method.
        if (isStatic && isFrozen) {
            Method[] methods = rootClass.getMethods();
            Callback[] fixedCallbacks = new Callback[methods.length];
            this.fixedInterceptorMap = CollectionUtils.newHashMap(methods.length);

            // TODO: small memory optimization here (can skip creation for methods with no advice)
            for (int x = 0; x < methods.length; x++) {
                Method method = methods[x];
                List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, rootClass);
                fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
                    chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
                this.fixedInterceptorMap.put(method, x);
            }

            // Now copy both the callbacks from mainCallbacks
            // and fixedCallbacks into the callbacks array.
            callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
            System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
            System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
            this.fixedInterceptorOffset = mainCallbacks.length;
        } else {
            callbacks = mainCallbacks;
        }
        return callbacks;
    }
```

最终会生成一个 DynamicAdvisedInterceptor，继续看 DynamicAdvisedInterceptor 的源码：
```Java
 @Override
        @Nullable
        public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
            Object oldProxy = null;
            boolean setProxyContext = false;
            Object target = null;
            TargetSource targetSource = this.advised.getTargetSource();
            try {
                if (this.advised.exposeProxy) {
                    // Make invocation available if necessary.
                    oldProxy = AopContext.setCurrentProxy(proxy);
                    setProxyContext = true;
                }
                // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
                target = targetSource.getTarget();
                Class<?> targetClass = (target != null ? target.getClass() : null);
                List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
                Object retVal;
                // Check whether we only have one InvokerInterceptor: that is,
                // no real advice, but just reflective invocation of the target.
                if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
                    // We can skip creating a MethodInvocation: just invoke the target directly.
                    // Note that the final invoker must be an InvokerInterceptor, so we know
                    // it does nothing but a reflective operation on the target, and no hot
                    // swapping or fancy proxying.
                    Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                    retVal = methodProxy.invoke(target, argsToUse);
                } else {
                    // We need to create a method invocation...
                    retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
                }
                retVal = processReturnType(proxy, target, method, retVal);
                return retVal;
            } finally {
                if (target != null && !targetSource.isStatic()) {
                    targetSource.releaseTarget(target);
                }
                if (setProxyContext) {
                    // Restore old proxy.
                    AopContext.setCurrentProxy(oldProxy);
                }
            }
        }
```

是不是非常眼熟？这就是前面提到过的，通过 CGLib 进行增强时，核心代码与 JDK 动态代理是非常类似的，而 `CglibMethodInvocation` 就是我们之前提到过的 `ReflectiveMethodInvocation`  类的子类，其执行逻辑几乎是一致的，在这里就不重复讲了。