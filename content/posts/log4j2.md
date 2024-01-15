---
title: 重新熟悉log4j配置
date: 2019-09-28 21:34:30
tags:
- log4j
categories:
- log4j
---

## 需求
如何自定义log4j的配置文件路径？

### 加载顺序
```
Automatic Configuration
Log4j has the ability to automatically configure itself during initialization. When Log4j starts it will locate all the ConfigurationFactory plugins and arrange then in weighted order from highest to lowest. As delivered, Log4j contains three ConfigurationFactory implementations: one for JSON, one for YAML, and one for XML.

Log4j will inspect the "log4j.configurationFile" system property and, if set, will attempt to load the configuration using the ConfigurationFactory that matches the file extension.
If no system property is set the YAML ConfigurationFactory will look for log4j2-test.yaml or log4j2-test.yml in the classpath.
If no such file is found the JSON ConfigurationFactory will look for log4j2-test.json or log4j2-test.jsn in the classpath.
If no such file is found the XML ConfigurationFactory will look for log4j2-test.xml in the classpath.
If a test file cannot be located the YAML ConfigurationFactory will look for log4j2.yaml or log4j2.yml on the classpath.
If a YAML file cannot be located the JSON ConfigurationFactory will look for log4j2.json or log4j2.jsn on the classpath.
If a JSON file cannot be located the XML ConfigurationFactory will try to locate log4j2.xml on the classpath.
If no configuration file could be located the DefaultConfiguration will be used. This will cause logging output to go to the console.
```

### Native Java Application
原生Java应用，通过启动时添加系统变量即可
```
-Dlog4j.configurationFile=log4j2.xml
```
### SpringBoot应用
SpringBoot应用也很简单
直接在application.properties中配置即可

```
logging.config=classpath:log4j2-hui-prod.xml
```

### Spring应用
网络上一搜，大部分都是这样的配置

```
    <context-param>
        <param-name>log4jConfigLocation</param-name>
        <param-value>classpath:log4j2-hui-prod.xml</param-value>
    </context-param>
    <context-param>
        <param-name>log4jRefreshInterval</param-name>
        <param-value>60000</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>
    </listener>
```
这样能行？log4j1 时代用的都是这套配置，但同样适用log4j2吗？追踪实现时发现Log4jConfigListener 在Spring 4.2.1版本后已经废弃了

先直接贴出答案，各位看官急用的话看到这里就可以解决问题。

#### 配置变更

log4jConfigLocation -> log4jConfiguration
```
    <context-param>
        <param-name>log4jConfiguration</param-name>
        <param-value>classpath:log4j2-hui-prod.xml</param-value>
    </context-param>
```

PS：大家是不是从来没有注意过web.xml下的web-app的配置，类似下面这样的：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee 
         http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">
        
</web-app>
```
修改为 3.1 or higher
可参考：http://www.oracle.com/webfolder/technetwork/jsc/xml/ns/javaee/index.html
```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
         http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
</web-app>
```

### 详细分析
#### web-app 这些配置是干嘛的
首先这是个 XML Schemas for Java EE Deployment Descriptors，主要是配置描述符

#### 根因分析
上面说到Spring实现的Listener已经废弃了，我们详细跟踪到了 Log4jServletContextListener ，这个是log4j2-web这个jar中实现的。按理说直接引入就应该可以达到修改配置路径的目的了，但发现实际上还是做了些限制，贴出下面的代码应该就能说明问题了

```java

package org.apache.logging.log4j.web;

import java.util.EnumSet;
import java.util.Set;
import javax.servlet.DispatcherType;
import javax.servlet.FilterRegistration;
import javax.servlet.ServletContainerInitializer;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;

import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.status.StatusLogger;

/**
 * In a Servlet 3.0 or newer environment, this initializer is responsible for starting up Log4j logging before anything
 * else happens in application initialization. For consistency across all containers, if the effective Servlet major
 * version of the application is less than 3.0, this initializer does nothing.
 */
public class Log4jServletContainerInitializer implements ServletContainerInitializer {

    private static final Logger LOGGER = StatusLogger.getLogger();

    @Override
    public void onStartup(final Set<Class<?>> classes, final ServletContext servletContext) throws ServletException {
        if (servletContext.getMajorVersion() > 2 && servletContext.getEffectiveMajorVersion() > 2 &&
                !"true".equalsIgnoreCase(servletContext.getInitParameter(
                        Log4jWebSupport.IS_LOG4J_AUTO_INITIALIZATION_DISABLED
                ))) {
            LOGGER.debug("Log4jServletContainerInitializer starting up Log4j in Servlet 3.0+ environment.");

            final FilterRegistration.Dynamic filter =
                    servletContext.addFilter("log4jServletFilter", Log4jServletFilter.class);
            if (filter == null) {
                LOGGER.warn("WARNING: In a Servlet 3.0+ application, you should not define a " +
                    "log4jServletFilter in web.xml. Log4j 2 normally does this for you automatically. Log4j 2 " +
                    "web auto-initialization has been canceled.");
                return;
            }

            final Log4jWebLifeCycle initializer = WebLoggerContextUtils.getWebLifeCycle(servletContext);
            initializer.start();
            initializer.setLoggerContext(); // the application is just now starting to start up

            servletContext.addListener(new Log4jServletContextListener());

            filter.setAsyncSupported(true); // supporting async when the user isn't using async has no downsides
            filter.addMappingForUrlPatterns(EnumSet.allOf(DispatcherType.class), false, "/*");
        }
    }
}

```
```java 
    /**
     * Returns the major version of the Servlet API that this
     * servlet container supports. All implementations that comply
     * with Version 3.0 must have this method return the integer 3.
     *
     * @return 3
     */
    public int getMajorVersion();
    
    
    /**
     * Returns the minor version of the Servlet API that this
     * servlet container supports. All implementations that comply
     * with Version 3.0 must have this method return the integer 0.
     *
     * @return 0
     */
    public int getMinorVersion();


    /**
     * Gets the major version of the Servlet specification that the
     * application represented by this ServletContext is based on.
     *
     * <p>The value returned may be different from {@link #getMajorVersion},
     * which returns the major version of the Servlet specification
     * supported by the Servlet container.
     *
     * @return the major version of the Servlet specification that the
     * application represented by this ServletContext is based on
     *
     * @throws UnsupportedOperationException if this ServletContext was
     * passed to the {@link ServletContextListener#contextInitialized} method
     * of a {@link ServletContextListener} that was neither declared in
     * <code>web.xml</code> or <code>web-fragment.xml</code>, nor annotated
     * with {@link javax.servlet.annotation.WebListener}
     *
     * @since Servlet 3.0
     */
    public int getEffectiveMajorVersion();
    
    
    /**
     * Gets the minor version of the Servlet specification that the
     * application represented by this ServletContext is based on.
     *
     * <p>The value returned may be different from {@link #getMinorVersion},
     * which returns the minor version of the Servlet specification
     * supported by the Servlet container.
     *
     * @return the minor version of the Servlet specification that the
     * application xrepresented by this ServletContext is based on
     *
     * @throws UnsupportedOperationException if this ServletContext was
     * passed to the {@link ServletContextListener#contextInitialized} method
     * of a {@link ServletContextListener} that was neither declared in
     * <code>web.xml</code> or <code>web-fragment.xml</code>, nor annotated
     * with {@link javax.servlet.annotation.WebListener}
     *
     * @since Servlet 3.0
     */
    public int getEffectiveMinorVersion();
```
简而言之就是：
2.5 => (major = 2 minor = 5)
3.1 => (major = 3 minor = 1)
MajorVersion => 实际引入jar的版本
EffectiveMajorVersion => web-app中声明的版本


## 参考
- http://www.voidcn.com/article/p-hzdnxyhq-pm.html
- https://www.oracle.com/webfolder/technetwork/jsc/xml/ns/javaee/index.html
- https://logging.apache.org/log4j/2.x/manual/configuration.html#AutomaticConfiguration