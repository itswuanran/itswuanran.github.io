---
title: 使用ApplicationContextAware接口
date: 2018-02-05 17:44:50
tags: 
- Spring
categories:
- Spring
---

## Aware接口

Aware接口为Spring容器的核心接口，是一个具有标识作用的超级接口，实现了该接口的bean是具有被Spring容器通知的能力，通知的方式是采用回调的方式。
Aware接口是一个空接口，实际的方法签名由各个子接口来确定，且该接口通常只会有一个接收单参数的set方法，该set方法的命名方式为set+去掉接口名中的Aware后缀，即XxxAware接口，则方法定义为setXxx()，例如BeanNameAware（setBeanName），ApplicationContextAware（setApplicationContext）。
Aware的子接口需要提供一个setXxx方法，我们知道set是设置属性值的方法，即Aware类接口的setXxx方法其实就是设置xxx属性值的。

Aware真正的含义是感知，Spring容器在初始化主动检测当前bean是否实现了Aware接口，如果实现了则回调其set方法将相应的参数设置给该bean，这个时候该bean就从Spring容器中取得相应的资源。

## ApplicationContextAware接口的作用

当一个类实现了这个接口（ApplicationContextAware）之后，这个类就可以获得ApplicationContext中的所有bean。
也就是说赋予了这个类获取其他bean的能力。

## 代码实现

```java
@Component
@Slf4j
public class SpringUtil implements ApplicationContextAware {
    private static final Logger LOG = LoggerFactory.getLogger(SpringUtil.class);
    private static ApplicationContext springContext;

    public SpringUtil() {
    }

    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        springContext = applicationContext;
    }

    public static <T> T getBean(String bean) {
        if (springContext == null) {
            LOG.error("springContext == null");
            throw new ApplicationRuntimeException("springContext == null");
        } else {
            boolean isExist = springContext.containsBean(bean);
            return isExist ? springContext.getBean(bean) : null;
        }
    }
}

```

## 风险点

```java
@Component
@Slf4j
public class TestSpringUtil {
    private static TestBean tb = SpringUtil.getBean("testBean");
    //...
}

```

当我们使用这个方式获取bean时，可能会出现TestBean先SpringUtil初始化，这时由于applicationContext会抛出空指针异常。
还有当一些组件为了扫描所有bean的注解使用Class.forName()获取类信息时也会导致同样的问题，不推荐使用声明静态变量这种方式。