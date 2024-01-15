---
title: 泛型
date: 2018-04-20 15:10:35
tags: [Java]
categories:
- Java
---

## 泛型
泛型是JDK 1.5的一项新特性，它的本质是参数化类型（Parameterized Type）的应用，也就是说所操作的数据类型被指定为一个参数，在用到的时候在指定具体的类型。这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口和泛型方法。

## 类型擦除
泛型信息只存在于代码编译阶段，在进入JVM之前，与泛型相关的信息会被擦除掉，专业术语叫做类型擦除。
通俗地讲，泛型类和普通类在java虚拟机内是没有什么特别的地方。
```java
List<String> l1 = new ArrayList<String>();
List<Integer> l2 = new ArrayList<Integer>();
System.out.println(l1.getClass() == l2.getClass());
```
结果为true，因为List<String>和List<Integer>在jvm中的Class都是 List.class。
泛型信息被擦除了。

Java的泛型类型擦除是发生在编译时的，在编译时会擦除泛型的参数化类型，并检查相应的代码，同时在相应的地方插入强制转换的代码。插入强制转换的代码可以通过反编译Java字节找到checkcast指令可知。
由于Java的泛型机制是在发生编译时擦除泛型的参数化类型，所以在JVM运行时不能获取到泛型参数化类型的基本信息。这点与C#的泛型底层机制有着明显的区别。

## 关于CHECKCAST指令
http://www.vmth.ucdavis.edu/incoming/Jasmin/ref--7.html

checkcast checks that the top item on the operand stack (a reference to an object or array) can be cast to a given type. For example, if you write in Java:
```java
return ((String)obj);
```
then the Java compiler will generate something like:

```
aload_1                        ; push -obj- onto the stack
checkcast java/lang/String     ; check its a String
areturn                        ; return it
```
checkcast is actually a shortand for writing Java code like:
```
if (! (obj == null  ||  obj instanceof <class>)) {
    throw new ClassCastException();
}
// if this point is reached, then object is either null, or an instance of
// <class> or one of its superclasses.
```
所以CHECKCAST指令实际上和INSTANCEOF指令是很像的，不同的是CHECKCAST如果失败，会抛出一个异常，INSTANCEOF是返回一个值。
http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.instanceof

The instanceof instruction is very similar to the checkcast instruction (§checkcast). It differs in its treatment of null, its behavior when its test fails (checkcast throws an exception,instanceof pushes a result code), and its effect on the operand stack.

另外，据虚拟机专家RednaxelaFX的说法，JVM有可能在运行时优化掉一些CHECKCAST指令。
http://hllvm.group.iteye.com/group/topic/25910

## 类型转换的时机
```java
    public <T> T get(StoreKey key);
    public <T> Map<StoreKey, T> multiGet(List<StoreKey> keys);
```
在使用multiGet时获取数据时，只有当遍历获取时才会触发类型cast。
Map<StoreKey, List<String>> rets = client.multiGet(keys);

执行这句是不会触发T的类型转换。
Map<StoreKey, CommonEntity> rets = client.multiGet(keys);
执行这个语句也不会触发类型转换，类型转换在使用时才会触发，捕获ClassCastException时要注意。

## 参考
- https://blog.csdn.net/hengyunabc/article/details/16358447