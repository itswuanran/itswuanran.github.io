---
title: java.lang.NoSuchMethodError排查
date: 2018-12-27 18:03:59
tags: 
- JVM
categories:
- JVM
---


## 代码案列
Rhino.java
```java
public class Rhino {
    public int someOps(int a, String b) {
        return 0;
    }
}
```
Async.java
```java
public class Async {
    public String doSomething(String b, String c) {
        Rhino rhino = new Rhino();
        rhino.someOps(0, "");
        return "normal";
    }
}
```
Clinet.java
```java
public class Client {
    public static void main(String[] args) {
        Async b = new Async();
        System.out.print(b.doSomething("a", "b"));
    }
}
```

```
⋊> test ls
Async.java  Client.java Rhino.java
⋊> test javac Client.java
⋊> test ls
Async.class  Async.java   Client.class Client.java  Rhino.class  Rhino.java
⋊> test javap -v Rhino.class
Classfile /Users/wuanran/Desktop/test/Rhino.class
  Last modified 2018-12-28; size 259 bytes
  MD5 checksum 3bf6201d461e5b960d606f5dbad16ad1
  Compiled from "Rhino.java"
public class Rhino
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#12         // java/lang/Object."<init>":()V
   #2 = Class              #13            // Rhino
   #3 = Class              #14            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               someOps
   #9 = Utf8               (ILjava/lang/String;)I
  #10 = Utf8               SourceFile
  #11 = Utf8               Rhino.java
  #12 = NameAndType        #4:#5          // "<init>":()V
  #13 = Utf8               Rhino
  #14 = Utf8               java/lang/Object
{
  public Rhino();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 2: 0

  public int someOps(int, java.lang.String);
    descriptor: (ILjava/lang/String;)I
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=3, args_size=3
         0: iconst_0
         1: ireturn
      LineNumberTable:
        line 5: 0
}
SourceFile: "Rhino.java"

## 修改Rhino.java，更改方法返回类型为String

    public String someOps(int a, String b) {
        return "";
    }

⋊> test javac Rhino.java
⋊> test javac Client.java
⋊> test java Client
Exception in thread "main" java.lang.NoSuchMethodError: Rhino.someOps(ILjava/lang/String;)I
	at Async.doSomething(Async.java:5)
	at Client.main(Client.java:5)

```

## java.lang.NoSuchMethodError分析
- 为什么会出现这个错误？
原因：Rhino.java中someOps方法在编译时候已经被虚拟机设置为(ILjava/lang/String;)I类型，而修改后someOps方法在class文件中为(ILjava/lang/String;)Ljava/lang/String;，因此异常发生。

- 这个例子在三方jar中也会遇到，A.jar B.jar C.jar，关系如下，C中一方法执行有如下依赖，B-0.1依赖A的0.1版本，C依赖B-0.1版本。当A升级到0.2版本后改变了原方法返回类型，此时引用的版本为： A-0.2 B-0.1，C中方法执行时会出现类似异常。

## 原因
在项目依赖比较复杂或者 Java 运行的环境有问题时，或者同一类型的 jar 包有不同版本存在，都可能触发该错误。本质上说是 JVM 找不到某个类的特定方法，也就是说 JVM 加载了错误版本的类。说白了，就是 JVM 找不到真正想要调用的方法啦！出现该错误的情形主要有以下两个种：

导入了不匹配的包版本；

开发环境和运行环境不一致。

## 解决方法
查看“External Libraries”，看报错的方法到底存不存在，如果不存在，说明这个包一定有问题啦，更新包就可以啦；如果存在，说明包已经引入成功，但集成开发环境有可能没有同步到，可以尝试强制更新的方法。此外，可以查看一下开发环境和运行环境是否一致，如果不一致，修改。

## 扩展
哪些方法的返回类型是在编译时确定，泛型又是如何处理的？