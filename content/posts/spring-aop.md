---
title: 理解SpringAOP
date: 2018-04-22 12:22:17
categories: 
- Spring
tags:
- Spring
---

## 代理模式

A->doSomething => A->Proxy->doSomething

## Aop基本概念

连接点（Joinpoint） ：程序能够应用通知的一个“时机”，这些“时机”就是连接点，例如方法被调用时、异常被抛出时等等。
通知（Advice） ：通知定义了切面是什么以及何时使用。描述了切面要完成的工作和何时需要执行这个工作。
切入点（Pointcut） ：通知定义了切面要发生的“故事”和时间，那么切入点就定义了“故事”发生的地点，例如某个类或方法的名称。
切面（Aspect） ：通知和切入点共同组成了切面，时间、地点和要发生的“故事”。
目标对象（Target Object） ：即被通知的对象。
AOP代理（AOP Proxy） 在Spring AOP中有两种代理方式，JDK动态代理和CGLIB代理。默认情况下，TargetObject实现了接口时，则采用JDK动态代理；反之，采用CGLIB代理。
织入（Weaving）把切面应用到目标对象来创建新的代理对象的过程，织入一般发生在如下几个时机：
1）编译时：当一个类文件被编译时进行织入，这需要特殊的编译器才能做到，例如AspectJ的织入编译器；
2）类加载时：使用特殊的ClassLoader在目标类被加载到程序之前增强类的字节代码；
3）运行时：切面在运行的某个时刻被织入，SpringAOP就是以这种方式织入切面的，原理是使用了JDK的动态代理。

## Spring AOP与AspectJ区别

Spring AOP 与ApectJ 的目的一致，都是为了统一处理横切业务，但与AspectJ不同的是，Spring AOP 并不尝试提供完整的AOP功能(即使它完全可以实现)，Spring AOP 更注重的是与Spring IOC容器的结合，并结合该优势来解决横切业务的问题，因此在AOP的功能完善方面，相对来说AspectJ具有更大的优势，同时，Spring注意到AspectJ在AOP的实现方式上依赖于特殊编译器(ajc编译器)，因此Spring很机智回避了这点，转向采用动态代理技术的实现原理来构建Spring AOP的内部机制（动态织入），这是与AspectJ（静态织入）最根本的区别。

在AspectJ 1.5后，引入@Aspect形式的注解风格的开发，Spring也非常快地跟进了这种方式，因此Spring 2.0后便使用了与AspectJ一样的注解。请注意，Spring 只是使用了与 AspectJ 5 一样的注解，但仍然没有使用 AspectJ 的编译器，底层依是动态代理技术的实现，因此并不依赖于 AspectJ 的编译器。

## Spring Aop类内部调用不会被切面的解决办法

首先要理解为什么不会被切面，Aop的实现是通过代理的方式。

```java
@Component
public class SomeServiceImpl implements SomeService  
{  
  
    public void someMethod()  
    {  
        someInnerMethod();  
        //foo...  
    }  
  
    public void someInnerMethod()  
    {  
        //bar...  
    }  
}
```

调用someService.someMethod();时内部调用someInnerMethod的切面并不会生效，因为someService此时表示的是代理对象。而在方法内部调用时表示的someService对象。

1. 修改类，不要出现“自调用”的情况：这是Spring文档中推荐的“最佳”方案；
2. 若一定要使用“自调用”，那么this.someInnerMethod()替换为：((CustomerService) AopContext.currentProxy()).someInnerMethod();
此时需要修改spring的aop配置：
<aop:aspectj-autoproxy expose-proxy="true" />
3. 静态织入，参见AspectJ。
4. InjectSelfAware，对那些有同一对象里存在方法间相互调用的类，提供一个父类，这个父类的目的是给所有它继承的子类提供一个字段self，self指向为代理对象。

## 动态代理解决问题的检查点：

- 需要AOP拦截的类是否是final的，final类不可使用CGLIB来代理。
- 是否在给BEAN配AOP的时候强制使用CGLIB，如果是则可指定proxyTargetClass属性以让spring强制代理目标类。
- 类是否被多次代理了，如果类被多次代理过，则第二次进行代理的时候拿到的是第一次代理后的对象，这个对象是个final形式的，所以会出现这个错误。

基于第三点要注意，类是否被多次代理不紧紧取决于类是否被配置了多次AOP，如果类实现了某个接口，则还要看类实现的接口是否被aop拦截过。如果类实现了接口且接口也被AOP拦截了，则很可能出现上面的错误（是否出错取决于AOP代理执行的顺序）。

## spring配置aop需要注意：

- 1、proxy-target-class属性值决定是基于接口的还是基于类的代理被创建，启动对@Aspectj的支持 true为cglib（基于类），false为jdk代理（基于接口），不写的话默认为false。为true的话，会导致拦截不了mybatis的mapper

```xml
<aop:aspectj-autoproxy proxy-target-class="false" />
```

2、在类没有实现任何接口，并且没有默认构造函数的情况下，通过构造函数注入时，目前的Spring是无法实现AOP切面拦截的。

## 参考

- <https://www.cnblogs.com/study-everyday/p/7429298.html>
- <http://blog.csdn.net/hupoling/article/details/54618461>
- <http://www.baeldung.com/spring-aop-vs-aspectj>
- <http://blog.csdn.net/javazejian/article/details/56267036>