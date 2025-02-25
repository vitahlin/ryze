---
tags:
  - Spring
  - SourceCode
---

## 关注问题

1. FactoryBean 的基本概念；
2. SmartFactoryBean 是什么；
3. FactoryBean 的实例化处理；
4. Bean 实例化的真正入口；
5. 单例 Bean 和原型 Bean 的实例化区别；

## 前言

上文中，我们讲到了 Bean 实例化的真正入口以及 Spring 对 BeanDefinition 合并的处理，我们已经得到一个合并后的 BeanDefinition，那么接下来就是根据合并后的 BeanDefinition 进行实例化。

## Bean 实例化入口

回顾一下 Bean 的实例化代码 `DefaultListableBeanFactory#preInstantiateSingletons`：
```Java
 @Override
    public void preInstantiateSingletons() throws BeansException {
        if (logger.isTraceEnabled()) {
            logger.trace("Pre-instantiating singletons in " + this);
        }

        /**
         * Iterate over a copy to allow for init methods which in turn register new bean definitions.
         * While this may not be part of the regular factory bootstrap, it does otherwise work fine.
         * 根据之前bean扫描得到bean名称集合生成一个新的bean名称数组
         */
        List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

        // Trigger initialization of all non-lazy singleton beans...
        for (String beanName : beanNames) {

            /**
             * 获取合并后的bean，自己本身是一个BeanDefinition，如果有父BeanDefinition，
             * 那么就是父BeanDefinition和当前BeanDefinition合并后生成一个新的RootBeanDefinition
             */
            RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);

            /**
             * 判断当前bean是不是单例、是不是非懒加载，
             * 所以这里只创建非懒加载的单例Bean。
             * isAbstract判断当前beanDefinition是不是抽象的，跟抽象类没有关系。
             * 抽象的BeanDefinition自己不能生成Bean，主要是给其他的beanDefinition当父亲的bean
             */
            if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                /*** 判断是不是FactoryBean */
                if (isFactoryBean(beanName)) {
                    /**
                     * 如果是一个factoryBean的话，先创建这个factoryBean
                     * 创建factoryBean时，需要在beanName前面拼接一个&符号
                     */
                    Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                    if (bean instanceof FactoryBean) {
                        FactoryBean<?> factory = (FactoryBean<?>) bean;
                        boolean isEagerInit;
                        if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                            isEagerInit = AccessController.doPrivileged(
                                (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                                getAccessControlContext());
                        } else {
                            /**
                             * 判断是否是一个SmartFactoryBean，并且不是懒加载的，就意味着,
                             * 在创建了这个factoryBean之后要立马调用它的getObject方法创建另外一个Bean
                             */
                            isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                        }
                        if (isEagerInit) {
                            getBean(beanName);
                        }
                    }
                } else {
                    /*** 不是factoryBean的话，则直接创建 */
                    getBean(beanName);
                }
            }
        }

        /**
         * 到这里，单例bean全部创建完成
         */

        /**
         * Trigger post-initialization callback for all applicable beans...
         * 在创建了所有的Bean之后，遍历为所有适用的 bean 触发初始化后回调，
         * 也就是这里会对延迟初始化的bean进行加载...
         */
        for (String beanName : beanNames) {
            /** 这一步其实是从缓存中获取对应的创建的Bean，这里获取到的必定是单例的 */
            Object singletonInstance = getSingleton(beanName);

            /**
             * 判断单例bean是不是实现了SmartInitializingSingleton接口,
             * 所有的单例bean都创建后再调用SmartInitializingSingleton
             */
            if (singletonInstance instanceof SmartInitializingSingleton) {
                /*** 记录执行时间，可以忽略 */
                StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize")
                    .tag("beanName", beanName);
                SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;

                /** spring安全管理器的判断，可以忽略 */
                if (System.getSecurityManager() != null) {
                    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }, getAccessControlContext());
                } else {
                    /**
                     * 调用了单例bean对象的afterSingletonsInstantiated方法,
                     * 所以afterSingletonsInstantiated这个方法的调用时间是所有的非懒加载单例bean创建之后
                     */
                    smartSingleton.afterSingletonsInstantiated();
                }
                smartInitialize.end();
            }
        }
    }
```

BeanDefinition 的合并前文分析过，接下来就是根据合并后的 BeanDefinition 来实例化 Bean。

看文中注释，最终实例化的方法是 getBean，调用 getBean 方法前会判断当前 Bean 是不是 FactoryBean。所以，本文源码的解析分为 2 部分：
1. FactoryBean 的判断逻辑分析；
2. Bean 实例化方法 getBean 的分析。


## FactoryBean 的判断逻辑

获取 `RootBeanDefinition` 完成后，接下来就调用 `getBean(beanName)` 进行 Bean 的实例化了，源码部分如下：
```java
if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {  
    /*** 判断是不是FactoryBean */  
    if (isFactoryBean(beanName)) {  
        /**  
         * 如果是一个factoryBean的话，先创建这个factoryBean  
         * 创建factoryBean时，需要在beanName前面拼接一个&符号  
         */  
        Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);  
        if (bean instanceof FactoryBean) {  
            FactoryBean<?> factory = (FactoryBean<?>) bean;  
            boolean isEagerInit;  
            if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {  
                isEagerInit = AccessController.doPrivileged(  
                    (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,  
                    getAccessControlContext());  
            } else {  
                /**  
                 * 判断是否是一个SmartFactoryBean，并且不是懒加载的，就意味着,  
                 * 在创建了这个factoryBean之后要立马调用它的getObject方法创建另外一个Bean  
                 */                
                 isEagerInit = (factory instanceof SmartFactoryBean &&  
                    ((SmartFactoryBean<?>) factory).isEagerInit());  
            }  
            if (isEagerInit) {  
                getBean(beanName);  
            }  
        }  
    } else {  
        /*** 不是factoryBean的话，直接创建就行了 */  
        getBean(beanName);  
    }  
}
```

当然这里的创建还是分了**工厂 Bean**和**非工厂 Bean**两个逻辑：
- 如果不是 FactoryBean 的话，那么很简单直接调用 `getBean` 方法获取实例化后的 Bean 对象；
- 如果是一个工厂 Bean，那么 `getBean(beanName)` 这一步只会创建一个工厂 Bean，接下来会通过 `isEagerInit` 参数判断是否需要初始化工厂 Bean 的对象，如果需要，再调用 `getBean(beanName)` 去立马获取工厂 Bean 需要生产的对象。我们可以新建测试类来验证。


### FactoryBean 介绍

`FactroyBean` 的作用是生产一个 `bean`，一般情况下，Spring 通过反射机制利用 `<bean>` 的 class 属性指定实现类实例化 Bean，在某些情况下，实例化 Bean 过程比较复杂，如果按照传统的方式，则需要在 `<bean>` 中提供大量的配置信息。配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。

Spring为此提供了一个 `org.springframework.bean.factory.FactoryBean` 的工厂类接口，用户可以通过实现该接口定制实例化bean的逻辑。`FactoryBean` 接口对于Spring框架来说占用重要的地位，Spring 自身就提供了70多个FactoryBean 的实现。

具体可以参考：[[Spring FactoryBean介绍]]


### 判断是否是 FactoryBean

了解 FactoryBean 相关概念后，继续来看 `AbstractBeanFactory#isFactoryBean` 方法，这里传入的参数是字符串类型的 baneName：
```java
@Override  
public boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException {  
    String beanName = transformedBeanName(name);  
    Object beanInstance = getSingleton(beanName, false);  
    if (beanInstance != null) {  
        return (beanInstance instanceof FactoryBean);  
    }  
     if (!containsBeanDefinition(beanName) && getParentBeanFactory() instanceof ConfigurableBeanFactory) {  
        return ((ConfigurableBeanFactory) getParentBeanFactory()).isFactoryBean(name);  
    }  
    return isFactoryBean(beanName, getMergedLocalBeanDefinition(beanName));  
}
```

首先 `Object beanInstance = getSingleton(beanName, false)` 直接从单例池中获取对应的 Bean 对象，看是否已经存在，如果已经存在的话判断是不是 FactoryBean。

如果不存在单例对象，只能通过 BeanDefinition 对象来判断了，`containsBeanDefinition(beanName)` 表示容器是否存在该 `BeanDefinition`  对象，` getParentBeanFactory() ` 则是获取父 `BeanFactory` 。

继续看 `isFactoryBean(beanName, getMergedLocalBeanDefinition(beanName))` 的逻辑，`AbstractBeanFactory#isFactoryBean` ：
```java
protected boolean isFactoryBean(String beanName, RootBeanDefinition mbd) {  
    Boolean result = mbd.isFactoryBean;  
    if (result == null) {  
        Class<?> beanType = predictBeanType(beanName, mbd, FactoryBean.class);  
        /** 判断是不是FactoryBean接口 */  
        result = (beanType != null && FactoryBean.class.isAssignableFrom(beanType));  
        mbd.isFactoryBean = result;  
    }  
    return result;  
}
```
`predictBeanType` 是根据 BeanDefinition 去拿 `beanClass` 的属性，就是拿到当前 BeanDefinition 对应的类，然后去判断是不是 FactoryBean 接口。

到这里就完成了 FactoryBean 的判断，接下来就来看当前 Bean 如果是 FactoryBean 那么应该怎么处理。

#### 当前是 FactoryBean

新建 FactoryBean 测试类：
```java
package org.springframework.vitahlin.bean.factory;  
  
public class Student {  
    private String name;  
    private String code;  
    public Student(String name, String code) {  
        this.name = name;  
        this.code = code;  
    }  
    public Student() {  
    }    @Override  
    public String toString() {  
        return "Student{}";  
    }  
}
```

```java
package org.springframework.vitahlin.bean.factory;  
  
import org.springframework.beans.factory.FactoryBean;  
import org.springframework.stereotype.Component;  
  
@Component  
public class StudentFactoryBean implements FactoryBean<Student> {  
    @Override  
    public Student getObject() throws Exception {  
        return new Student();  
    }  
    @Override  
    public Class<?> getObjectType() {  
        return null;  
    }  
    @Override  
    public String toString() {  
        return "StudentFactoryBean{}";  
    }  
}
```


首先来看当前是 FactoryBean 的情况：
```java
if (isFactoryBean(beanName)) {  
    Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);  
    if (bean instanceof FactoryBean) {  
        FactoryBean<?> factory = (FactoryBean<?>) bean;  
        boolean isEagerInit;  
        if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {  
            isEagerInit = AccessController.doPrivileged(  
                (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,  
                getAccessControlContext());  
        } else {  
             isEagerInit = (factory instanceof SmartFactoryBean &&  
                ((SmartFactoryBean<?>) factory).isEagerInit());  
        }  
        if (isEagerInit) {  
            getBean(beanName);  
        }  
    }
```

这里先调用 `getBean(FACTORY_BEAN_PREFIX + beanName)` 方法创建 FactoryBean 对象，**注意这里先创建的是 FactoryBean 对象，** 新增断点如图所示：
![image.png](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2023/202301102343254.png)

可以看到当前的 `beanName` 为 `studentFactoryBean`，创建出来的 Object 对象是类型是 `StudentFactoryBean`。


##### 判断是否是 SmartFactoryBean

接着会继续判断 `FactoryBean` 是否实现了接口 `SmartFactoryBean`，并且判断方法 `isEagerInit` 的值，值为 true 表示当前对象需要在 Spring 启动时候就创建，直接调用 getBean 方法获取对应的 Bean 对象。

#### FactoryBean 处理流程

所以当 Bean 是一个 FactoryBean 的时候，流程如下：
1. 传入一个特殊的 FactoryBean 的 BeanName；
2. 调用 `getBean`  方法得到一个实例化后的 FactoryBean 对象；
3. 判断 FactoryBean 对象是否实现了接口 SmartFactoryBean
    - 如果是则调用 `getObject()`  方法立马进行 FactoryBean 生产的对象创建；
    - 如果不是则跳过。



## getBean

处理完 FactoryBean 的判断，Spring 就会直接调用 `getBean`  方法来获取一个实例化的 Bean 对象。这也就是 Bean 最终实例化的逻辑，跟进代码，可以看到，这里最终调用的是  `AbstractBeanFactory#getBean` ：
```java
@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}
```

可以看到，这里是直接调用了方法 `doGetBean`，`doGetBean`  方法源码如下：
```Java
  protected <T> T doGetBean(
        String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
        throws BeansException {
        
        /**
         * 转换BeanName
         * 处理传入的名字，因为传进来的可能是一个Bean的别名，也可能是一个FactoryBean；
         * 1. 首先去除factoryBean的修饰符，比如name="&instA" -----> instA;
         * 2. 取指定的alias所表示的最终的beanName，比如传入的是别名ia指向为instA的bean，那么返回的是instA
         */
        String beanName = transformedBeanName(name);
        Object beanInstance;
        
        /**
         * Eagerly check singleton cache for manually registered singletons.
         * 先从单例池子里面尝试获取这个bean，在这里它根本不会判断这个bean是单例的还是原型的，
         * 因为绝大多数bean都是单例的，所以即使是原型bean，再多取一次也无妨
         */
        Object sharedInstance = getSingleton(beanName);
        
        if (sharedInstance != null && args == null) {
            if (logger.isTraceEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
                } else {
                    logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            
            /**
             * 如果直接从单例池中获取到了这个 bean(sharedInstance),我们能直接返回吗？
             * 当然不能，因为获取到的Bean可能是一个factoryBean,
             * 如果我们传入的name是&+beanName这种形式的话，
             * 那是可以返回的，但是我们传入的更可能是一个beanName，
             * 那么这个时候Spring就还需要调用这个sharedInstance的getObject方法来创建真正被需要的Bean.
             *
             * 如果是工厂类且name不以&开头，则调用工厂类的getObject()
             * 其他情况返回原对象
             *
             * 从 beanInstance 中获取公开的Bean对象，主要处理beanInstance是FactoryBean对象的情况，
             * 如果不是FactoryBean会直接返回beanInstance实例
             */
            beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }
        
        /*** 这里，就是单例bean的创建逻辑 */
        else {
            /**
             * Fail if we're already creating this bean instance:
             * We're assumably within a circular reference.
             * 在缓存中获取不到这个bean，原型下的循环依赖直接报错。
             * 非主线代码，到时候循环依赖部分再讲解。
             */
            if (isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }
            
            /**
             * Check if bean definition exists in this factory.
             * 获取父BeanFactory。
             * 找不到我们就从父容器中再找一次,
             * 我们简单的示例是不会有父容器存在的，这一块可以理解为递归到父容器中查找，
             * 跟在当前容器查找逻辑是类似的
             */
            BeanFactory parentBeanFactory = getParentBeanFactory();
            /**
             * 如果存在父BeanFactory并且当前的BeanFactory里面没有这个BeanDefinition
             */
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                /**
                 * Not found -> check parent.
                 * 这里就去调用父BeanFactory的getBean()方法，
                 * 这段if逻辑就是尝试从父Bean工厂里面去拿，看能不能拿到
                 */
                String nameToLookup = originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                        nameToLookup, requiredType, args, typeCheckOnly);
                } else if (args != null) {
                    // Delegation to parent with explicit args.
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                } else if (requiredType != null) {
                    // No args -> delegate to standard getBean method.
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                } else {
                    return (T) parentBeanFactory.getBean(nameToLookup);
                }
            }
            
            /**
             * 如果不仅仅是为了类型推断，也就是代表我们要对进行实例化
             * 那么就将bean标记为正在创建中，其实就是将这个beanName放入到alreadyCreated这个set集合中
             */
            if (!typeCheckOnly) {
                markBeanAsCreated(beanName);
            }
            
            /*** 监控代码运行时间的，不是主要逻辑，忽略 */
            StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
                .tag("beanName", name);
            
            try {
                if (requiredType != null) {
                    beanCreation.tag("beanType", requiredType::toString);
                }
                
                /**
                 * 要创建bean，这里就是先根据beanName拿到合并之后的BeanDefinition。
                 * 为什么这里需要再获取一次，因为经过之前的操作，RootBeanDefinition可能已经发生了改变，
                 * 其中的stale属性可能已经设为 true，这时需要去容器里重新获取，而不是直接从缓存中返回。
                 * 例如上面的markBeanAsCreated()方法就会修改stale属性
                 */
                RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                
                /**
                 * 检查合并后的bd是否是abstract，如果是直接报错。
                 * 但是在这个检查现在已经没有作用了，必定会通过
                 */
                checkMergedBeanDefinition(mbd, beanName, args);
                
                /**
                 * Guarantee initialization of beans that the current bean depends on.
                 *
                 * 这里拿到BeanDefinition对象中的dependOn属性，这是一开始注册BeanDefinition的时候设置的。
                 * DependsOn注解标注的当前这个Bean所依赖的bean名称的集合，
                 * 在创建当前这个Bean前，必须要先将其依赖的Bean先完成创建
                 */
                String[] dependsOn = mbd.getDependsOn();
                if (dependsOn != null) {
                    /*** 遍历所有声明的依赖 */
                    for (String dep : dependsOn) {
                        /**
                         * 判断beanName是不是被dep依赖了，如果是，则出现了循环依赖，抛出异常。
                         * spring无法解决由dependsOn注解引出的循环依赖
                         */
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        
                        /*** 注册bean跟其依赖的依赖关系，key为依赖，value为依赖所从属的bean */
                        registerDependentBean(dep, beanName);
                        try {
                            /**
                             * 在这里，调用getBean就又执行了当前方法，只不过参数不一样，
                             * 会去创建依赖的bean，如果创建失败则抛出异常
                             * 比如userService依赖orderService，那么在这里就会去创建orderService
                             */
                            getBean(dep);
                        } catch (NoSuchBeanDefinitionException ex) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                        }
                    }
                }
                
                /**
                 * 执行到这里，依赖的bean已经创建出来了，就开始创建当前的bean
                 * 正常的流程，需要实例化，还需要属性注入，还需要类加载
                 */
                
                /**
                 * create bean instance.
                 * 在这里，创建Bean实例。创建时，需要判断bean是不是单例的，还是原型的，还是其他的类型
                 */
                if (mbd.isSingleton()) {
                    /*** 如果是单例bean，那么就有2个步骤，一个是创建，创建完成后还要放到单例池里面去 */
                    sharedInstance = getSingleton(beanName, () -> {
                        try {
                            return createBean(beanName, mbd, args);
                        } catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.
                            destroySingleton(beanName);
                            throw ex;
                        }
                    });
                    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                } else if (mbd.isPrototype()) {
                    /**
                     * It's a prototype -> create a new instance.
                     * 原型bean的处理
                     */
                    Object prototypeInstance = null;
                    try {
                        /**
                         * 如果是原型bean，那么就直接创建了
                         * beforePrototypeCreation先记录下当前bean正在创建
                         * 然后createBean就是创建bean
                         */
                        beforePrototypeCreation(beanName);
                        prototypeInstance = createBean(beanName, mbd, args);
                    } finally {
                        /**
                         * 原型bean创建完成后，就去掉表示正在创建的标记
                         */
                        afterPrototypeCreation(beanName);
                    }
                    beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                } else {
                    /**
                     * 既不是单例bean也不是原型bean的处理，实现其他作用域的处理
                     * 比如request scope，表示同一个请求内的bean是同一个
                     * session scope，表示同一个session内的bean是同一个
                     *
                     * 以request为例，会有一个request.getAttribute(beanName)方法，先调用这个方法获取bean
                     * 如果不存在，就创建一个bean，并且steAttribute(bean)
                     * session则类似
                     */
                    String scopeName = mbd.getScope();
                    if (!StringUtils.hasLength(scopeName)) {
                        throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
                    }
                    
                    /**
                     * 拿到Scope的实现类
                     */
                    Scope scope = this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }
                    try {
                        /**
                         * 拿到Scope的实现类后，去看get方法，这里以SessionScope为例
                         */
                        Object scopedInstance = scope.get(beanName, () -> {
                            beforePrototypeCreation(beanName);
                            try {
                                /*** createBean实际跳转到AbstractAutowireCapableBeanFactory */
                                return createBean(beanName, mbd, args);
                            } finally {
                                afterPrototypeCreation(beanName);
                            }
                        });
                        beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    } catch (IllegalStateException ex) {
                        throw new ScopeNotActiveException(beanName, scopeName, ex);
                    }
                }
            } catch (BeansException ex) {
                beanCreation.tag("exception", ex.getClass().toString());
                beanCreation.tag("message", String.valueOf(ex.getMessage()));
                cleanupAfterBeanCreationFailure(beanName);
                throw ex;
            } finally {
                beanCreation.end();
            }
        }
        
        return adaptBeanInstance(name, beanInstance, requiredType);
    }
```

代码较长，截图如下：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202210141643939.png)

大致分为5个部分：
1. transformedBeanName 对名称进行转换；
2. getSingleton 从单例池容器缓存中获取 bean；
3. if 代码块，从缓存中获取到了 Bean 进行后续的操作；
4. else 代码块，缓存中没有则进行 Bean 的创建；
5. adaptBeanInstance 进行类型转换。

### transformedBeanName-处理Bean的名称

这一步是处理 Bean 的名称，传进来一个 BeanName 有可能是别名，也有可能是 FactoryBean 的名称。在这里统一处理 BeanName，跟进方法 `AbstractBeanFactory#transformedBeanName` ：
```java
protected String transformedBeanName(String name) {  
    /**  
     * 先调用transformedBeanName去除前缀，  
     * 再调用canonicalName获取别名  
     */  
    return canonicalName(BeanFactoryUtils.transformedBeanName(name));  
}
```

这里是调用 `transformedBeanName` 处理 FactoryBean 的情况，再调用 `canonicalName`  方法处理别名。

先来看 `BeanFactoryUtils#transformedBeanName` ：
```java
public static String transformedBeanName(String name) {  
    Assert.notNull(name, "'name' must not be null");  
    /*** FACTORY_BEAN_PREFIX = "&"，不包含前缀直接返回 */  
    if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {  
        return name;  
    }  
    return transformedBeanNameCache.computeIfAbsent(name, beanName -> {  
        /*** 循环处理，带有多个&的都会去掉 */  
        do {  
            beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());  
        }  
        while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));  
        return beanName;  
    });  
}
```
判断是否是 `FactoryBean`  名称的格式，如果不是直接返回，如果是则处理前缀。

再看方法 `SimpleAliasRegistry#canonicalName` ：
```java
public String canonicalName(String name) {  
    String canonicalName = name;  
    /**  
     * Handle aliasing...     
     * 处理别名  
     */  
    String resolvedName;  
    do {  
        /*** 通过aliasMap获取BeanName，直到不再有别名 */  
        resolvedName = this.aliasMap.get(canonicalName);  
        /*** 可能存在 A->B->C->D, 所以需要一直循环到没有别名为止 */  
        if (resolvedName != null) {  
            canonicalName = resolvedName;  
        }  
    }  
    while (resolvedName != null);  
    return canonicalName;  
}
```

这里确定最终的名称，一个 Bean 可能会定义很多别名，这个时候需要确定最根本的那个 name，用这个最根本的 name 来作为 BeanName 去进行后续的操作。这里同样有个循环去处理，因为别名也会有多重，会存在别名的别名这种情况。

### getSingleton-从单例池中获取单例Bean

在上一步中我们已经获取到了真正的 BeanName，那么接下来，就可以利用这个 `BeanName`  到容器的缓存中尝试获取 Bean。如果之前已经创建过，这里就可以直接获取到 Bean。

这里的缓存包括三级，但是**这三级缓存并不是包含的关系，而是一种互斥的关系**，一个 Bean 无论处于何种状态，它在同一时刻只能处于某个缓存当中。
这个缓存是 Spring 中用来处理循环依赖的，这里我们就简单理解为，**当前就是根据 BeanName 从单例池中获取对应的单例 Bean，详细的源码后续在讲解 Spring 循环依赖的处理中介绍。**

**这里即使它是原型Bean，也会调用getSingleton这个方法。** Spring 中默认大部分 Bean 为单例 Bean。

### 缓存池中已经存在Bean的处理

如果直接从单例池中获取到了这个 bean(sharedInstance)，我们能直接返回吗？  
> 如果我们传入的 name 是 `&+beanName`  这种形式的话，那是可以返回的，因为这表示当前需要的 FactoryBean 对象。
> 但是我们传入的更可能是一个 beanName，就需要做额外的判断，如果是普通 Bean 那直接返回，如果是 FactoryBean 则需要调用调用 getObject 方法来创建真正被需要的 Bean。

```java
if (sharedInstance != null && args == null) {  
    if (logger.isTraceEnabled()) {  
        if (isSingletonCurrentlyInCreation(beanName)) {  
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +  
                "' that is not fully initialized yet - a consequence of a circular reference");  
        } else {  
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");  
        }  
    }  
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);  
}
```

if 语句进去，最终执行的方法是 `getObjectForBeanInstance` ，跟进去可以看到这里会进行一些类型的判断，会尝试从缓存获取，最后会调用 `getObjectFromFactoryBean` 方法从 `FactoryBean` 里获取实例对象。

`AbstractBeanFactory#getObjectForBeanInstance` ：
```java
protected Object getObjectForBeanInstance(  
    Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {  
    /**  
     * 如果指定的name是工厂相关（以&为前缀）且 beanInstance又不是FactoryBean类型则验证不通过  
     */  
    if (BeanFactoryUtils.isFactoryDereference(name)) {  
        if (beanInstance instanceof NullBean) {  
            return beanInstance;  
        }  
        if (!(beanInstance instanceof FactoryBean)) {  
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());  
        }  
        if (mbd != null) {  
            mbd.isFactoryBean = true;  
        }  
        return beanInstance;  
    }  
    /**  
     * 现在我们有了 bean 实例，它可能是普通的 bean 或 FactoryBean。  
     * 如果它是一个 FactoryBean，我们使用它来创建一个 bean 实例，除非调用者实际上想要一个对工厂的引用。  
     * 如果是普通bean，直接返回了  
     */  
    if (!(beanInstance instanceof FactoryBean)) {  
        return beanInstance;  
    }  
    /*** 加载factoryBean */  
    Object object = null;  
    if (mbd != null) {  
        mbd.isFactoryBean = true;  
    } else {  
        /*** 尝试从缓存中获取 */  
        object = getCachedObjectForFactoryBean(beanName);  
    }  
    if (object == null) {  
        /**  
         * 到这里可以确定 beanInstance 一定是 FactoryBean 类型  
         */  
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;  
        /**  
         * Caches  object obtained from FactoryBean if it is a singleton.         
         * 如果是单例，则缓存从 FactoryBean 获得的对象。  
         * containsBeanDefinition 检测 beanDefinitionMap 中也就是在所有已经加载的类中检测是否定义 beanName  
         */        
         if (mbd == null && containsBeanDefinition(beanName)) {  
            /**  
             * 将存储XML文件的 GenericBeanDefinition 转换为 RootBeanDefinition，  
             * 如果指定的 beanName 是子 bean 的话会同时合并父类的相关属性  
             */  
            mbd = getMergedLocalBeanDefinition(beanName);  
        }  
        /*** 是否是用用户自己定义的还是用程序本身定义的 */  
        boolean synthetic = (mbd != null && mbd.isSynthetic());  
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);  
    }  
    return object;  
}
```


```Java
if (!(beanInstance instanceof FactoryBean)) {  
    return beanInstance;  
}  
```
这里判断 Bean 的类型是不是 FactoryBean，如果不是就是普通的 Bean ，并且缓存已经存在不需要再执行一次 Bean 的创建流程，直接返回缓存的对象即可。否则，继续往下执行，说明当前的 Bean 是 FactoryBean。

#### FactoryBean 创建 Bean 的流程

首先尝试缓存中获取， `FactoryBeanRegistrySupport#getCachedObjectForFactoryBean`：
```Java
    @Nullable
    protected Object getCachedObjectForFactoryBean(String beanName) {
        return this.factoryBeanObjectCache.get(beanName);
    }
```
`factoryBeanObjectCache` 就是 `FactoryBean` 创建的 `bean` 的缓存，创建一次后会进行缓存，下次直接拿。

缓存中不存在，则肯定得执行创建逻辑了，跟进源码  `FactoryBeanRegistrySupport#getObjectFromFactoryBean` ：
```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {  
    /*** 如果是单例 */  
    if (factory.isSingleton() && containsSingleton(beanName)) {  
        /*** 加锁，保证单例 */  
        synchronized (getSingletonMutex()) {  
            /*** 先从缓存中获取 */  
            Object object = this.factoryBeanObjectCache.get(beanName);  
            if (object == null) {  
                /*** 从factoryBean中获取Bean */  
                object = doGetObjectFromFactoryBean(factory, beanName);  
                // Only post-process and store if not put there already during getObject() call above  
                // (e.g. because of circular reference processing triggered by custom getBean calls)                
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);  
                if (alreadyThere != null) {  
                    object = alreadyThere;  
                } else {  
                    if (shouldPostProcess) {  
                        if (isSingletonCurrentlyInCreation(beanName)) {  
                            // Temporarily return non-post-processed object, not storing it yet..  
                            return object;  
                        }  
                        beforeSingletonCreation(beanName);  
                        try {  
                            /*** 调用ObjectFactory的后置处理器 */  
                            object = postProcessObjectFromFactoryBean(object, beanName);  
                        } catch (Throwable ex) {  
                            throw new BeanCreationException(beanName,  
                                "Post-processing of FactoryBean's singleton object failed", ex);  
                        } finally {  
                            afterSingletonCreation(beanName);  
                        }  
                    }  
                    if (containsSingleton(beanName)) {  
                        this.factoryBeanObjectCache.put(beanName, object);  
                    }  
                }  
            }  
            return object;  
        }  
    } else {  
        /*** 从FactoryBean获取对象 */  
        Object object = doGetObjectFromFactoryBean(factory, beanName);  
        if (shouldPostProcess) {  
            try {  
                /*** 后置处理 FactoryBean 创建的对象 */  
                object = postProcessObjectFromFactoryBean(object, beanName);  
            } catch (Throwable ex) {  
                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);  
            }  
        }  
        return object;  
    }  
}
```

代码比较简单，其中方法 `doGetObjectFromFactoryBean` 就是 FactoryBean 创建 bean 的过程。创建出来的 Bean 还需要判断是不是有后置处理器，如果有的话就执行 Bean 的后置处理器，即方法 `postProcessObjectFromFactoryBean` 。如果不是单例 Bean，再次创建就是了，流程也一样。


### 真正进入创建Bean的流程

经过前面这么多铺垫，才真正走到了创建 Bean 的地方，即 else 的代码块。这里会比较复杂且啰嗦，需要点耐心看完。

```java
if (isPrototypeCurrentlyInCreation(beanName)) {  
    throw new BeanCurrentlyInCreationException(beanName);  
}
```
一开始，先对原型类型的循环依赖进行校验，原型 Bean 出现循环依赖直接抛异常，中断流程，不继续往下走。


```java
BeanFactory parentBeanFactory = getParentBeanFactory();  
 if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {  
    String nameToLookup = originalBeanName(name);  
    if (parentBeanFactory instanceof AbstractBeanFactory) {  
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(  
            nameToLookup, requiredType, args, typeCheckOnly);  
    } else if (args != null) {  
        // Delegation to parent with explicit args.  
        return (T) parentBeanFactory.getBean(nameToLookup, args);  
    } else if (requiredType != null) {  
        // No args -> delegate to standard getBean method.  
        return parentBeanFactory.getBean(nameToLookup, requiredType);  
    } else {  
        return (T) parentBeanFactory.getBean(nameToLookup);  
    }  
}
```
这部分代码，主要处理的是存在父 BeanFactory，并且当前的 BeanFactory 里面没有这个 BeanDefinition，就尝试从父 BeanFactory 中获取，如果有对应 Bean 对象就直接返回了。


```java
if (!typeCheckOnly) {  
    markBeanAsCreated(beanName);  
}
```
因为接下来，我们要开始创建 Bean 了，所以这里先把这个 Bean 标记为正在创建中，跟进方法`markBeanAsCreated` ，`AbstractBeanFactory#markBeanAsCreated` ：
```java
protected void markBeanAsCreated(String beanName) {  
    if (!this.alreadyCreated.contains(beanName)) {  
        synchronized (this.mergedBeanDefinitions) {  
            if (!this.alreadyCreated.contains(beanName)) {  
                // Let the bean definition get re-merged now that we're actually creating  
                // the bean... just in case some of its metadata changed in the meantime.                clearMergedBeanDefinition(beanName);  
                this.alreadyCreated.add(beanName);  
            }  
        }  
    }  
}
```
其实就是将当前这个 beanName 放入到  `alreadyCreated`  这个集合中。


接着，处理被 `@DependsOn` 注解标注的依赖，比如 userService 依赖 orderService，那么在这里就会去创建 orderService ：
```java
String[] dependsOn = mbd.getDependsOn();  
if (dependsOn != null) {  
    /*** 遍历所有声明的依赖 */  
    for (String dep : dependsOn) {  
        /**  
         * 判断beanName是不是被dep依赖了，如果是，则出现了循环依赖，抛出异常。  
         * spring无法解决由dependsOn注解引出的循环依赖  
         */  
        if (isDependent(beanName, dep)) {  
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");  
        }  
        /*** 注册bean跟其依赖的依赖关系，key为依赖，value为依赖所从属的bean */  
        registerDependentBean(dep, beanName);  
        try {  
            /**  
             * 在这里，调用getBean就又执行了当前方法，只不过参数不一样，  
             * 会去创建依赖的bean，如果创建失败则抛出异常  
             */            
             getBean(dep);  
        } catch (NoSuchBeanDefinitionException ex) {  
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);  
        }  
    }  
}
```


创建依赖的Bean对象后，我们就可以创建当前所需要的 Bean 对象。这里又对 Bean 的类型进行判断，区分三种不同的创建逻辑：
1. 单例Bean
2. 原型Bean
3. 既不是单例Bean也不是原型Bean的其他作用域的Bean

#### 原型Bean的创建

先来看原型 Bean 的创建：
```java
else if (mbd.isPrototype()) {  
    Object prototypeInstance = null;  
    try {  
        beforePrototypeCreation(beanName);  
        prototypeInstance = createBean(beanName, mbd, args);  
    } finally {  
        afterPrototypeCreation(beanName);  
    }  
    beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);  
}
```

原型 Bean 的创建流程比较简单：
1. `beforePrototypeCreation` 方法先标记下当前的 Bean 正在创建中；
2. 调用 `createBean(beanName, mbd, args)` 进行创建 Bean，createBean 的流程后续详细分析，这里直接理解为就是创建一个 Bean；
3. afterPrototypeCreation 去掉表示正在创建的标记；
4. getObjectForBeanInstance，从 beanInstannce 中获取公开的 Bean 对象，主要处理 beanInstance 是 FactoryBean 对象的情况，如果不是 FactoryBean 会直接返回 beanInstance 实例。

#### 既不是单例 Bean 也不是原型 Bean的处理

```java
 else {  
    String scopeName = mbd.getScope();  
    if (!StringUtils.hasLength(scopeName)) {  
        throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");  
    }  
    Scope scope = this.scopes.get(scopeName);  
    if (scope == null) {  
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");  
    }  
    try {  
        Object scopedInstance = scope.get(beanName, () -> {  
            beforePrototypeCreation(beanName);  
            try {  
                return createBean(beanName, mbd, args);  
            } finally {  
                afterPrototypeCreation(beanName);  
            }  
        });  
        beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);  
    } catch (IllegalStateException ex) {  
        throw new ScopeNotActiveException(beanName, scopeName, ex);  
    }  
}
```
这里先拿到 Scope 的实现类，然后这里是一段 lambda 代码，实际上调用的 Scope 实现类的 `get`  方法。

我们以 `SessionScope`  为例，跟进 `SessionScope#get`  方法：
```java
@Override  
public Object get(String name, ObjectFactory<?> objectFactory) {  
   Object mutex = RequestContextHolder.currentRequestAttributes().getSessionMutex();  
   synchronized (mutex) {  
      return super.get(name, objectFactory);  
   }
}
```
参数中 name 是 Bean 的名字，**objectFactory 就是创建 Bean 的 lambda 表达式** 。

跟进 `super.get`  方法 `AbstractRequestAttributesScope#get` ：
```java
@Override  
public Object get(String name, ObjectFactory<?> objectFactory) {  
    RequestAttributes attributes = RequestContextHolder.currentRequestAttributes();  
    /**  
     * name是bean的名字，这里有一个getAttribute方法，  
     * 看它的实现ServletRequestAttributes  
     */    
    Object scopedObject = attributes.getAttribute(name, getScope());  
    if (scopedObject == null) {  
        /**  
         * 如果没有拿到，执行传进来的第二个参数lambda表达式，创建一个bean对象  
         */  
        scopedObject = objectFactory.getObject();  
        /*** 创建完成后，再set进去 */  
        attributes.setAttribute(name, scopedObject, getScope());  
        // Retrieve object again, registering it for implicit session attribute updates.  
        // As a bonus, we also allow for potential decoration at the getAttribute level.        
        Object retrievedObject = attributes.getAttribute(name, getScope());  
        if (retrievedObject != null) {  
            // Only proceed with retrieved object if still present (the expected case).  
            // If it disappeared concurrently, we return our locally created instance.            
            scopedObject = retrievedObject;  
        }  
    }  
    return scopedObject;  
}
```

继续跟进，最终执行的方法是 `ServletRequestAttributes#getAttribute` ：
```java
@Override  
public Object getAttribute(String name, int scope) {  
    if (scope == SCOPE_REQUEST) {  
        /*** 如果是request，就直接调用getAttribute */  
        if (!isRequestActive()) {  
            throw new IllegalStateException(  
                "Cannot ask for request attribute - request is not active anymore!");  
        }  
        return this.request.getAttribute(name);  
    } else {  
        /**  
         * 如果是session，先调用session，再调用getAttribute  
         */        
        HttpSession session = getSession(false);  
        if (session != null) {  
            try {  
                Object value = session.getAttribute(name);  
                if (value != null) {  
                    this.sessionAttributesToUpdate.put(name, value);  
                }  
                return value;  
            } catch (IllegalStateException ex) {  
                // Session invalidated - shouldn't usually happen.  
            }  
        }  
        return null;  
    }  
}
```

到这里，我们就能串联起整个流程。如果是 SessionScope，同一 session 中的 Bean 都是一样的，Bean 先从 HttpSession 中获取，如果不存在则同样执行 createBean 进行创建，创建完成后把 Bean 放到集合中去。


#### 单例Bean的创建

分析完原型 Bean 和其他作用域类型 Bean 的创建流程，就到了最重要的单例 Bean 创建的环节，这也是创建 Bean 的核心代码：
```java
if (mbd.isSingleton()) {  
    sharedInstance = getSingleton(beanName, () -> {  
        try {  
            return createBean(beanName, mbd, args);  
        } catch (BeansException ex) {  
            destroySingleton(beanName);  
            throw ex;  
        }  
    });  
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);  
}
```

#### getSingleton

这是一段 lambda 表达式，我们先来看 getSingleton 方法：
```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {  
    Assert.notNull(beanName, "Bean name must not be null");  
    synchronized (this.singletonObjects) {  
        Object singletonObject = this.singletonObjects.get(beanName);  
        if (singletonObject == null) {  
            if (this.singletonsCurrentlyInDestruction) {  
                throw new BeanCreationNotAllowedException(beanName,  
                    "Singleton bean creation not allowed while singletons of this factory are in destruction " +  
                        "(Do not request a bean from a BeanFactory in a destroy method implementation!)");  
            }  
            if (logger.isDebugEnabled()) {  
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");  
            }  
            beforeSingletonCreation(beanName);  
            boolean newSingleton = false;  
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);  
            if (recordSuppressedExceptions) {  
                this.suppressedExceptions = new LinkedHashSet<>();  
            }  
            try {  
                singletonObject = singletonFactory.getObject();  
                newSingleton = true;  
            } catch (IllegalStateException ex) {  
                singletonObject = this.singletonObjects.get(beanName);  
                if (singletonObject == null) {  
                    throw ex;  
                }  
            } catch (BeanCreationException ex) {  
                if (recordSuppressedExceptions) {  
                    for (Exception suppressedException : this.suppressedExceptions) {  
                        ex.addRelatedCause(suppressedException);  
                    }  
                }  
                throw ex;  
            } finally {  
                if (recordSuppressedExceptions) {  
                    this.suppressedExceptions = null;  
                }  
                afterSingletonCreation(beanName);  
            }  
            if (newSingleton) {  
                addSingleton(beanName, singletonObject);  
            }  
        }  
        return singletonObject;  
    }  
}
```

1. 这个方法的流程并不复杂，首先 `Object singletonObject = this.singletonObjects.get(beanName)`  从单例池中获取 Bean 对象，如果不为空则直接返回，为空表示 Bean 还未创建，继续执行创建流程。
2.  `beforeSingletonCreation(beanName)` 则是在 Bean 创建前先打个标记，表示当前这个 Bean 正在创建。
3. `singletonObject = singletonFactory.getObject()`  ，`getSingleton`  这个方法的第二个参数 singletonFactory 实际上是一个 lambda 表达式，所以这里就会去执行 lambda 表达式，执行完创建一个对象。
4. `afterSingletonCreation(beanName)` 在单例完成创建后，将 beanName 从 singletonsCurrentlyInCreation 中移除，标志着这个单例已经完成了创建。
5. 创建完成后，就通过 `addSingleton(beanName, singletonObject)` 加入到单例池中去，下次直接获取即可。

我们可以来看下 addSingleton 源码 DefaultSingletonBeanRegistry.addSingleton：
```java
protected void addSingleton(String beanName, Object singletonObject) {  
    synchronized (this.singletonObjects) {  
        this.singletonObjects.put(beanName, singletonObject);  
        this.singletonFactories.remove(beanName);  
        this.earlySingletonObjects.remove(beanName);  
        this.registeredSingletons.add(beanName);  
    }  
}
```
就是把创建出来的 Bean 对象放到缓存池 `singletonObjects` 中去。

这就是 Bean 的创建流程，步骤 3 中的 lambda 其实是调用了 `createBean` 的方法，后续我们将继续分析 `createBean` 的源码。

## 总结

本文分析了 Bean 实例化流程中的 getBean 方法，介绍了 FactoryBean 的处理以及单例 Bean 、原型 Bean，以及其他作用域类型的 Bean 的创建流程，最终 Bean 的创建都是通过 lambda 方式调用了 `createBean` 方法，后续将进一步介绍 `createBean` 的流程。