---
title: maven编译遇到的bug
date: 2020-05-30 22:04:05
tags:
- maven
categories:
- maven
---

## 问题
最近在使用maven过程中发现一个bug，`maven` 3.6.3 版本在解决嵌套属性访问时存在空指针异常

## issues
官方提交的链接
https://issues.apache.org/jira/browse/MNG-6921

## maven源码debug思路
执行mvnDebug命令，执行后会看到监听一个端口，然后下载maven源代码配置到idea remote debug 端口，就可以断点调试了
