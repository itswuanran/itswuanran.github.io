---
title: SSM单机项目构建
date: 2018-01-20 19:39:35
categories:
- Spring
tags:
- Spring
- mybatis
---
## 从零开始构建SSM项目
Spring & SpringMVC & mybatis 项目构建
## 脚手架选择
- mybatis example 代码生成配置
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <!--
        出现错误：Caused by: java.lang.ClassNotFoundException: com.mysql.jdbc.Driver
        解决办法：将本地的MAVEN仓库中的mysql驱动引入进来
    -->
    <context id="mysqlgenerator" targetRuntime="MyBatis3">
        <!--不生成注释-->
        <commentGenerator>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>
        <!-- 配置数据库连接 -->

        <!--<jdbcConnection driverClass="com.mysql.jdbc.Driver"-->
        <!--connectionURL="jdbc:mysql://localhost:3306/quake"-->
        <!--userId="root"-->
        <!--password="root"/>-->

        <!-- 指定javaBean生成的位置 -->
        <javaModelGenerator targetPackage="com.anruence.entity" targetProject="src/main/java">
            <!-- 在targetPackage的基础上，根据数据库的schema再生成一层package，最终生成的类放在这个package下，默认为false -->
            <property name="enableSubPackages" value="true"/>
            <!-- 设置是否在getter方法中，对String类型字段调用trim()方法 -->
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <!--指定sql映射文件生成的位置 -->
        <sqlMapGenerator targetPackage="sqlmap" targetProject="src/main/resources">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>
        <!-- 指定dao接口生成的位置，mapper接口 -->
        <javaClientGenerator type="XMLMAPPER" targetPackage="com.anruence.dao" targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <!-- table表生成对应的DoaminObject -->
        <!--<table tableName="R_DEVICES"/>-->
        <table tableName="quake"/>

    </context>

</generatorConfiguration>
```

## SpringMVC实战
- SpringMVC注解
- 理解RESTful
- 各项配置
- 捕获5XX异常，@ControllerAdvice
- CORS支持 注解 or xml
- 异步请求
## 字段参数验证（自定义Validator）
- 使用hibernate-validator，自定义注解
- @Max、@Length等
## 图片上传（非OSS）
- tomcat 配置上传文件至不同路径下防止图片上传到war包目录，服务重启时图片丢失。

## 拦截器 & Aspect应用
- Aop搭配注解，记录日志，限制请求等。

## quartz定时任务
- cron任务
- 进程内任务

## swagger-ui 集成
- 集成依赖
- 可访问路径配置

### 各注解详解
- @Api
- @ApiOpreator
- @ApiModel

## mybatis应用

### mybatis sqlmap自动生成
- 使用maven插件自动生成CRUD

### mybatis example试用
- example 多条件查询，类似于ActiveRecord。

### mybatis-plus集成
- 构建查询语句（ActiveRecord）

## 构建jenkins发布平台

### 集成代码平台，tomcat
- jenkins部署服务
- github & tomcat user

## maven profile环境区分
- profile区分环境使用不同配置
```
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.0.0</version>
                <configuration>
                    <warName>${artifactId}-${env}-${version}</warName>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-deploy-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
        </plugins>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <excludes>
                    <exclude>product/*</exclude>
                    <exclude>qa/*</exclude>
                </excludes>
            </resource>
            <resource>
                <directory>src/main/resources/${active.env}</directory>
                <targetPath>config/spring/local</targetPath>
            </resource>
        </resources>
    </build>
    <profiles>
        <profile>
            <id>test</id>
            <activation>
                <activeByDefault>true</activeByDefault>
                <property>
                    <name>env</name>
                    <value>test</value>
                </property>
            </activation>
            <properties>
                <active.env>qa</active.env>
            </properties>
        </profile>
        <profile>
            <id>product</id>
            <activation>
                <property>
                    <name>env</name>
                    <value>product</value>
                </property>
            </activation>
            <properties>
                <active.env>product</active.env>
            </properties>
        </profile>
    </profiles>
```
关键在于targetPath属性的设置。

