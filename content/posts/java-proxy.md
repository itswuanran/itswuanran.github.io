---
title: Aop中的代理限制
date: 2018-04-10 17:11:23
categories: 
- Spring
tags: 
- cglib
---
## 从一个异常说起
```
Caused by: org.springframework.beans.factory.BeanNotOfRequiredTypeException: Bean named 'merchantCouponModule' is expected to be of type 'com.xxx.OrderServiceImpl' but was actually of type 'com.sun.proxy.$Proxy96'
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:378)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202)
	at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.autowireResource(CommonAnnotationBeanPostProcessor.java:522)
	at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.getResource(CommonAnnotationBeanPostProcessor.java:496)
	at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor$ResourceElement.getResourceToInject(CommonAnnotationBeanPostProcessor.java:627)
	at org.springframework.beans.factory.annotation.InjectionMetadata$InjectedElement.inject(InjectionMetadata.java:169)
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:88)
	at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessPropertyValues(CommonAnnotationBeanPostProcessor.java:318)
	... 60 more
```
报错表示注入类型不正确，期望类型是OrderServiceImpl，而实际类型是com.sun.proxy（JDK代理对象）。

## JDK代理 & Cglib

```
	/**
	 * Construct a new JdkDynamicAopProxy for the given AOP configuration.
	 * @param config the AOP configuration as AdvisedSupport object
	 * @throws AopConfigException if the config is invalid. We try to throw an informative
	 * exception in this case, rather than let a mysterious failure happen later.
	 */
	public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
		Assert.notNull(config, "AdvisedSupport must not be null");
		if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("No advisors and no TargetSource specified");
		}
		this.advised = config;
	}


	@Override
	public Object getProxy() {
		return getProxy(ClassUtils.getDefaultClassLoader());
	}

	@Override
	public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
```

我们知道proxy是JDK的代理对象，通常为了对接口进行增强，常用于AOP中。
我们看下Jdk代理的Aop实现，Proxy.newProxyInstance生成的Proxy对象是一个实现了proxiedInterfaces接口类型为$Proxy?的实例，即为异常中的$Proxy96。
然而由于我们在注入bean时使用的是：
```java
@Resource
private OrderServiceImpl orderService;
```
@Autowire和@Resources都可以实现根据名称和类型注入，但是会检测声明变量的类型的（checkBeanNotOfRequiredType），没法强制转换。
所以生成的代理对象和原对象不匹配，抛出以上异常。但是如何解决呢？

## 解决方案

## 方案一
虽然代理对象和原对象不匹配，但是接口匹配，所以我们可以使用：
```java
@Resource
private OrderService orderService;
```

## 方案二
因为JDK的代理会生成$Proxy?的实例，可以使用CgLib代替JDK的代理。CgLib可以生成目标类的子类来实现增强，代理对象依然是OrderServiceImpl类型的。引入Aspectj，设置proxy-target-class=true即可。
```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>
```
注意：如果实现多个接口时不会使用cglib代理，参考hasNoUserSuppliedProxyInterfaces。
```java
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface()) {
				return new JdkDynamicAopProxy(config);
			}
			if (!cglibAvailable) {
				throw new AopConfigException(
						"Cannot proxy target class because CGLIB2 is not available. " +
						"Add CGLIB to the class path or specify proxy interfaces.");
			}
			return CglibProxyFactory.createCglibProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
	/**
	 * Determine whether the supplied {@link AdvisedSupport} has only the
	 * {@link org.springframework.aop.SpringProxy} interface specified
	 * (or no proxy interfaces specified at all).
	 */
	private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class[] interfaces = config.getProxiedInterfaces();
		return (interfaces.length == 0 || (interfaces.length == 1 && SpringProxy.class.equals(interfaces[0])));
	}

```