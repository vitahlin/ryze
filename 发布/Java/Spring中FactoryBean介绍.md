---
title: Spring中FactoryBean介绍
slug: spring-factorybean-introduction
date: 2024-09-07
categories:
  - Source Code
tags:
  - Spring
featuredImage: https://sonder.vitah.me/featured/e8962fa8cb5b633b5d8219478d105259.webp
desc: 在 Spring 框架中，FactoryBean 是一个特殊的 Bean，它并不直接返回实例，而是通过实现 getObject 方法来生成或修饰对象。本文介绍了 FactoryBean 的概念及其与 BeanFactory 的区别，探讨了它在 Spring 容器中的重要作用，并举例说明如何使用 FactoryBean 来简化复杂对象的实例化过程。
draft: false
---

# 概述

Spring 容器中有两种 bean：普通 bean 和工厂 bean。

Spring 直接使用前者，`FactoryBean` 跟普通 Bean 不同，其返回的对象不是指定类的一个实例，而是该 `FactoryBean` 的 `getObject` 方法所返回的对象。

我们可以通过 `BeanPostProcessor` 来干涉 Spring 创建 Bean 的过程，但是如果我们想一个 Bean 完完全全由我们来创造，也是可以的，比如通过 `FactoryBean` 接口。

`FactoryBean` 接口对于 Spring 框架来说占有重要的地位，Spring 自身就提供了很多 `FactoryBean` 的实现。它们隐藏了实例化一些复杂 bean 的细节，给上层应用带来了便利。从 Spring 3.0 开始， `FactoryBean` 开始支持泛型，即接口声明改为 `FactoryBean<T>` 的形式。

# FactoryBean 与 BeanFactory 区别

`FactoryBean` 翻译过来是工厂 Bean，`BeanFactory` 翻译过来是 Bean 工厂，`FactoryBean` 是 Bean 工厂 `beanFactory` 中的一个 Bean，只不过这个 Bean 和一般的 `bean` 不一样，它有着自己的特殊之处，特殊在什么地方呢？**这个 Bean 不是简单的 Bean，而是一个能生产或者修饰对象生成的工厂 Bean，它的实现与设计模式中的工厂模式和修饰器模式类似。**

# FactoryBean 接口源码

接口源码：

```java
package org.springframework.beans.factory;

import org.springframework.lang.Nullable;

public interface FactoryBean<T> {

    String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

    @Nullable
    T getObject() throws Exception;

    @Nullable
    Class<?> getObjectType();

    default boolean isSingleton() {
        return true;
    }
}
```

- getObject：返回由 FactoryBean 创建的 bean 实例，如果 `isSingleton()` 返回 true，则该实例会放到 Spring 容器的单实例缓存池中。`getObject` 方法内部由开发者自己去实现对象的创建，然后将创建好的对象返回给 Spring 容器；
- getObjectType：返回 `FactoryBean` 创建的 bean 类型；
- isSingleton：返回由 `FactoryBean` 创建的 bean 实例的作用域是 `singleton` 还是 `prototype`，如果返回 false，表示由这个 FactoryBean 创建的对象是多例的，那么我们每次从容器中 `getBean` 的时候都会去重新调用 FactoryBean 中的 `getObject` 方法获取一个新的对象。若返回 true，表示创建的对象是单例的，那么我们每次从容器中获取这个对象的时候都是同一个对象。

`FactoryBean` 接口很简单，就提供了三个方法 `getObject`、`getObjectType`、`isSingleton`。就是这三个方法却成为了 Spring 中很多功能的基础，搜索整个 Spring 的源码可以找到很多 `FactoryBean`，除了 Spring 自身提供的以外，在和一些框架进行集成的时候，同样有 `FactoryBean` 的影子，比如和 mybatis 集成的时候的 `SqlSessionFactoryBean`。

# SmartFactoryBean

SmartFactoryBean 是 FactoryBean 的子接口，对 FactoryBean 进行了扩展。

```java
public interface SmartFactoryBean<T> extends FactoryBean<T> {

    default boolean isPrototype() {
        return false;
    }

    default boolean isEagerInit() {
        return false;
    }
}
```

方法 `isPrototype`：FactoryBean 管理的对象是否为 `prototype`；

### isEagerInit

容器启动时是否初始化 FactoryBean 管理的对象。

**标准的 FactoryBean 不需要急于初始化，它的 `getObject()` 只会在实际访问时被调用，即使是在单例对象的情况下也是如此。**

如果需要在容器启动时同步创建 Bean 实例**可以重写 `isEagerInit()`，true 表示容器创建阶段就会调用 `getObject()` 去实例化对应的 Bean 实例**。

# FactoryBean 的使用

示例代码：

```java
public class Student {
    private String name;
    private String code;

    public Student(String name, String code) {
        this.name = name;
        this.code = code;
    }

    public Student() {
    }

    @Override
    public String toString() {
        return "Student{}";
    }
}
```

```java
@Component
public class MyFactoryBean implements FactoryBean<Student> {

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
        return "MyFactoryBean{}";
    }
}
```

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanScanConfig.class);
AnnotatedBeanDefinitionReader annotatedBeanDefinitionReader = new AnnotatedBeanDefinitionReader(context);

annotatedBeanDefinitionReader.register(MyFactoryBean.class);

Student stu = (Student) context.getBean("myFactoryBean");

MyFactoryBean myFactoryBean = (MyFactoryBean) context.getBean("&myFactoryBean");
System.out.println("stu:" + stu);
System.out.println("myFactoryBean:" + myFactoryBean);
```

运行结果：

```
stu:Student{}
myFactoryBean:MyFactoryBean{}
```

打印结果很奇怪，通过 `myFactoryBean` 获得了 `Student` 对象，通过 `&myFactoryBean` 获得了 `MyFactoryBean` 对象。为什么会这样？

这就是 FactoryBean 的神奇，通俗点讲 FactoryBean 是一个工厂 Bean，它是一种可以生产 Bean 的 Bean，通过其 getObejct 方法生产 Bean。当然不论是 FactoryBean 还是 FactoryBean 生产的 Bean 都是受 Spring 管理的，不然通过 getBean 方法是拿不到的。

`FactroyBean` 的作用是生产一个 `bean`，这里有一个疑问 Spring 就是用来生产 `bean` 和管理 `bean` 的，为什么还要有 `FactoryBean`？

一般情况下，Spring 通过反射机制利用的 class 属性指定实现类实例化 Bean，在某些情况下，实例化 Bean 过程比较复杂，如果按照传统的方式，则需要在中提供大量的配置信息。配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring 为此提供了一个 org. springframework. bean. factory. FactoryBean 的工厂类接口，用户可以通过实现该接口定制实例化 bean 的逻辑。FactoryBean 接口对于 Spring 框架来说占用重要的地位，Spring 自身就提供了 70 多个 FactoryBean 的实现。

FactoryBean 的真正目的是让开发者自己去定义那些复杂的 bean 并交给 spring 管理，如果 bean 中要初始化很多变量，而且要进行许多操作，那么使用 spring 的自动装配是很难完成的，所以 Spring 的开发者把这些工作交给了我们。