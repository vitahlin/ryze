---
tags:
    - Spring
    - SourceCode
---

## 关注问题

1. Bean 实例化入口；[[#^c29b20]]
2. BeanDefinition 为什么需要合并；

## 前言

前文分析了 Spring 的 Bean 扫描流程的源码，我们已经得到了一个 BeanDefinition 集合，那么很显然，接下来就是遍历 BeanDefinition 列表，进行 Bean 的实例化。

## Spring上下文初始化

首先，来看 Spring 中容器的初始化：
```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {  
    this();  
    register(componentClasses);  
    refresh();  
}
```

其中，Bean 的实例化流程是在 `refresh()`  方法中处理的，跟进方法 `AnnotationConfigApplicationContext#refresh`：
```java
@Override  
public void refresh() throws BeansException, IllegalStateException {  
    synchronized (this.startupShutdownMonitor) {  
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");  
        prepareRefresh();  
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
        prepareBeanFactory(beanFactory);  
        try {  
            postProcessBeanFactory(beanFactory);  
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");  
            
            invokeBeanFactoryPostProcessors(beanFactory);  
            
            registerBeanPostProcessors(beanFactory);  
            beanPostProcess.end();  
            initMessageSource();  
            initApplicationEventMulticaster();  
             onRefresh();  
            registerListeners();  
            
            finishBeanFactoryInitialization(beanFactory);  
            
            finishRefresh();  
        } catch (BeansException ex) {  
            if (logger.isWarnEnabled()) {  
                logger.warn("Exception encountered during context initialization - " +  
                    "cancelling refresh attempt: " + ex);  
            }  
            destroyBeans();  
            cancelRefresh(ex);  
            throw ex;  
        } finally {  
            contextRefresh.end();  
        }  
    }  
}
```

其中，Bean 的实例化是在 `finishBeanFactoryInitialization(beanFactory)` 中完成的，`AbstractApplicationContext#finishBeanFactoryInitialization` 源码如下：
```java
 protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
            beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
        }
        
        if (!beanFactory.hasEmbeddedValueResolver()) {
            beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
        }
        
        String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
        for (String weaverAwareName : weaverAwareNames) {
            getBean(weaverAwareName);
        }
        
        beanFactory.setTempClassLoader(null);
        
        beanFactory.freezeConfiguration();
        
        beanFactory.preInstantiateSingletons();
    }
```

生成非懒加载的单例 Bean 的处理逻辑在最后一行 `beanFactory.preInstantiateSingletons()`，实际上调用的是 `DefaultListableBeanFactory#preInstantiateSingletons`，看这方法命名就知道，实际上这个方法处理了单例 Bean 的实例化。

## preInstantiateSingletons-Bean 实例化的入口

^c29b20

跟进源码 `DefaultListableBeanFactory#preInstantiateSingletons` 来看实例化的流程：
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
             *
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

代码较长，截图如下：
![image.png](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2023/202301102210871.png)

这里主要有3部分逻辑，分别对应其中三个箭头：
1. BeanDefnition 的合并-getMergedLocalBeanDefinition；
2. BeanDefnition 的实例化-getBean；
3. SmartInitializingSingleton 接口的实现。

接下来，我们就来逐步看这三个步骤，本文重点介绍 BeanDefinition 的合并流程。

## BeanDefinition 合并-getMergedLocalBeanDefinition

根据之前扫描得到 Bean 名称集合生成一个新的 Bean 名称数组，然后逐个遍历开始 Bean 的实例化。

首先，就是根据 `BeanDefinition`  生成 `RootBeanDefinition`  对象：
```java
RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
```

在 Spring 中，一个 BeanDefinition 是可以设置父 BeanDefinition 的，仅需要调用其 `setParentName`   即可，之所以出现父子 Bean 是因为 Spring 允许将相同 Bean 的定义给抽出来，成为一个父 BeanDefinition，这样其它的 BeanDefinition 就能共用这些公共的数据了，**Bean 有层次关系，子类需要合并父类的属性方法**。Spring 中支持父子 BeanDefinition，和 Java 父子类类似，但是完全不是一回事。

例如：
```xml
<bean id="parent" class="com.vitah.service.Parent" scope="prototype"/>
<bean id="child" class="com.vitah.service.Child"/>
```
这么定义的情况下，child 是单例 Bean。

```xml
<bean id="parent" class="com.vitah.service.Parent" scope="prototype"/>
<bean id="child" class="com.vitah.service.Child" parent="parent"/>
```
但是这么定义的情况下，child 就是原型 Bean 了。

因为 child 的父 BeanDefinition 是 parent，所以会继承 parent 上所定义的 scope 属性。

Bean 定义公共的抽象类是 `AbstractBeanDefinition` ，普通的 Bean 在 Spring 加载 Bean 定义的时候，实例化出来的是 `GenericBeanDefinition` ，而 Spring 上下文包括实例化所有 Bean 用的 AbstractBeanDefinition 是 RootBeanDefinition ，这时候就使用 `getMergedLocalBeanDefinition` 方法做了一次转化，将非 `RootBeanDefinition` 转换为 ` RootBeanDefinition` 以供后续操作。

跟进方法 `AbstractBeanFactory#getMergedLocalBeanDefinition` :
```java
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
		RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
		if (mbd != null && !mbd.stale) {
			return mbd;
		}
		return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
	}
```
`mergedBeanDefinitions` 集合会先判断合并之后的 BeanDefinition 是否已经存在了，存在了直接返回。

不存在则继续跟进方法 `getMergedBeanDefinition` ，最终调用的方法是 `AbstractBeanFactory#getMergedBeanDefinition` ：
```java
protected RootBeanDefinition getMergedBeanDefinition(
			String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
			throws BeanDefinitionStoreException {

		synchronized (this.mergedBeanDefinitions) {
			/*** 准备一个RootBeanDefinition变量引用，用于记录要构建和最终要返回的BeanDefinition */
			RootBeanDefinition mbd = null;
			RootBeanDefinition previous = null;

			// Check with full lock now in order to enforce the same merged instance.
			if (containingBd == null) {
				mbd = this.mergedBeanDefinitions.get(beanName);
			}

			if (mbd == null || mbd.stale) {
				previous = mbd;
				/*** 判读有没有指定parent */
				if (bd.getParentName() == null) {
					/**
					 * Use copy of given root bean definition.
					 * 如果自己就是一个RootBeanDefinition，那么就克隆一下，生成一个新的RootBeanDefinition
					 */
					if (bd instanceof RootBeanDefinition) {
						mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
					} else {
						mbd = new RootBeanDefinition(bd);
					}
				}
				else {
					/**
					 * Child bean definition: needs to be merged with parent.
					 * 定义父亲的BeanDefinition
					 */
					BeanDefinition pbd;
					try {
						/*** 拿到这个bean的parent的名字 */
						String parentBeanName = transformedBeanName(bd.getParentName());
						
						/*** 当前名字不等于parentBean的名字，递归处理，一直往上找 */
						if (!beanName.equals(parentBeanName)) {
							pbd = getMergedBeanDefinition(parentBeanName);
						} else {
							BeanFactory parent = getParentBeanFactory();
							if (parent instanceof ConfigurableBeanFactory) {
								pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
							} else {
								throw new NoSuchBeanDefinitionException(parentBeanName,
									"Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
										"': cannot be resolved without a ConfigurableBeanFactory parent");
							}
						}
					} catch (NoSuchBeanDefinitionException ex) {
						throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
							"Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
					}
					// Deep copy with overridden values.
					/*** 生成一个新的BeanDefinition，并且覆盖掉父亲的属性 */
					mbd = new RootBeanDefinition(pbd);
					mbd.overrideFrom(bd);
				}

				// Set default singleton scope, if not configured before.
				if (!StringUtils.hasLength(mbd.getScope())) {
					mbd.setScope(SCOPE_SINGLETON);
				}

				// A bean contained in a non-singleton bean cannot be a singleton itself.
				// Let's correct this on the fly here, since this might be the result of
				// parent-child merging for the outer bean, in which case the original inner bean
				// definition will not have inherited the merged outer bean's singleton status.
				if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
					mbd.setScope(containingBd.getScope());
				}

				// Cache the merged bean definition for the time being
				// (it might still get re-merged later on in order to pick up metadata changes)
				if (containingBd == null && isCacheBeanMetadata()) {
					/*** 放到map中 */
					this.mergedBeanDefinitions.put(beanName, mbd);
				}
			}
			if (previous != null) {
				copyRelevantMergedBeanDefinitionCaches(previous, mbd);
			}
			return mbd;
		}
	}
```


### 没有父BeanDefinition的情况

源码如下：
```java
if (bd.getParentName() == null) {  
    /**  
     * Use copy of given root bean definition.     
     * 如果自己就是一个RootBeanDefinition，那么就克隆一下，生成一个新的RootBeanDefinition  
     */    
    if (bd instanceof RootBeanDefinition) {  
        mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();  
    } else {  
        mbd = new RootBeanDefinition(bd);  
    }  
}
```

如果一个 BeanDefinition 没有设置 `parentName` ，表示该 BeanDefinition 是没有父 BeanDefinition 的。
此时，如果这个 BeanDefinition 是 RootBeanDefinition 的实例或子类实例，Spring就会直接克隆一个一模一样的 BeanDefinition，`cloneBeanDefinition` 方法的代码很简单，即 `new RootBeanDefinition(this)` 。 
如果不是 RootBeanDefinition 类型，则创建一个 RootBeanDefinition 并将当前 bd 合并进去。

这时候肯定会有疑问，两个合并的代码不是一模一样吗？
其实不是，如果一个 BeanDefinition 是 RootBeanDefinition 的子类，那么调用克隆方法的时候，就会 new 一个该子类，然后开始克隆，比如 `new ConfigurationClassBeanDefinition(this)`。

### 有父BeanDefinition的情况

```java
 else {  
    /**  
     * Child bean definition: needs to be merged with parent.     
     * 定义父亲的BeanDefinition  
     */
    BeanDefinition pbd;  
    try {  
        /*** 拿到这个bean的parent的名字 */  
        String parentBeanName = transformedBeanName(bd.getParentName());  
        /*** 当前名字不等于parentBean的名字，递归处理，一直往上找 */  
        if (!beanName.equals(parentBeanName)) {  
            pbd = getMergedBeanDefinition(parentBeanName);  
        } else {  
            BeanFactory parent = getParentBeanFactory();  
            if (parent instanceof ConfigurableBeanFactory) {  
                pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);  
            } else {  
                throw new NoSuchBeanDefinitionException(parentBeanName,  
                    "Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +  
                        "': cannot be resolved without a ConfigurableBeanFactory parent");  
            }  
        }  
    } catch (NoSuchBeanDefinitionException ex) {  
        throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,  
            "Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);  
    }  
    // Deep copy with overridden values.  
    /*** 生成一个新的BeanDefinition，并且覆盖掉父亲的属性 */  
    mbd = new RootBeanDefinition(pbd);  
    mbd.overrideFrom(bd);  
}
```

如果一个 `BeanDefinition` 有父类，那么 Spring 就会获取到父类的 `BeanName` (因为可能存在别名, 所以调用了 `transformedBeanName方法` )。
如果父类的 `BeanName` 和当前  `BeanName` 不一样，说明是存在真正的父类的。这个判断是用来防止我们将当前 `BeanName` 设置为当前 `BeanDefinition` 的父类而导致死循环的。
如果真正存在父类，Spring 就会先将父类合并了，因为父类可能还有父类，该递归方法结束后就能获取到一个完整的父 `BeanDefinition`  了，然后 ` new `  了一个 ` RootBeanDefinition ` ，将完整的父 `BeanDefinition`  放入进去，从而初步完成了合并。

执行后，我们就得到了**最终的合并之后的 BeanDefinition，放到了 Map 对象 `mergedBeanDefinitions` 中，后续就从 `mergedBeanDefinitions` 拿合并后的 BeanDefiniton 进行实例化。**


## 结语

主要介绍了 BeanDefinition 的合并流程，后续继续分析 Spring 如何根据合并后的 BeanDefinition 来得到 Bean 对象。