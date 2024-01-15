---
title: HashMap序列化的一个问题
date: 2018-01-23 14:46:19
tags: 
- HashMap
categories:
- Java
---
## 测试代码

```
public class SimpleSerializationTest {
    @Test
    public void testHashMap() throws Exception {
        // 注意这句
        HashMap<String, String> hmap = new HashMap<String, String>() {{
            put("key1", "value1");
            put("key2", "value2");
        }};

        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutput out = null;
        out = new ObjectOutputStream(bos);
        out.writeObject(hmap);
        byte[] yourBytes = bos.toByteArray();
        if (out != null) {
            out.close();
        }
        bos.close();

        ByteArrayInputStream bis = new ByteArrayInputStream(yourBytes);
        ObjectInput in = null;
        in = new ObjectInputStream(bis);
        Object o = in.readObject();
        bis.close();
        if (in != null) {
            in.close();
        }
        assertEquals(hmap, o);
    }
}
```
## 问题解释

```
The exception message tells you exactly what the problem is: you are trying to serialize an instance of class SimpleSerializationTest, and that class is not serializable.

Why? Well, you have created an anonymous inner class of SimpleSerializationTest, one that extends HashMap, and you are trying to serialize an instance of that class. Inner classes always have references to the relevant instance of their outer class, and by default, serialization will try to traverse those.

I observe that you use a double-brace {{ ... }} syntax as if you think it has some sort of special significance. It is important to understand that it is actually two separate constructs. The outer pair of braces appearing immediately after a constructor invocation mark the boundaries of the inner class definition. The inner pair bound an instance initializer block, such as you can use in any class body (though they are unusual in contexts other than anonymous inner classes). Ordinarily, you would also include one or more method implementations / overrides inside the outer pair, either before or after the initializer block.
```

意思是 我们创建了一个继承HashMap的SimpleSerializationTest匿名内部类，然后试着去序列化这个内部类的实例。内部类持有外部类的引用，默认情况下序列化会序列化内部类和外部类。

继承一个类一般引用自身，不会有父类的引用。
根源在于双大括号的语法上，外部大括号创建了一个匿名类，内部大括号指定的是匿名类初始化代码块。

## 扩展引申

## 什么是Java中的内存泄露
导致内存泄漏主要的原因是，先前申请了内存空间而忘记了释放。如果程序中存在对无用对象的引用，那么这些对象就会驻留内存，消耗内存，因为无法让垃圾回收器GC验证这些对象是否不再需要。如果存在对象的引用，这个对象就被定义为"有效的活动"，同时不会被释放。要确定对象所占内存将被回收，我们就要务必确认该对象不再会被使用。典型的做法就是把对象数据成员设为null或者从集合中移除该对象。但当局部变量不需要时，不需明显的设为null，因为一个方法执行完毕时，这些引用会自动被清理。
在Java中，内存泄漏就是存在一些被分配的对象，这些对象有下面两个特点，首先，这些对象是有被引用的，即在有向树形图中，存在树枝通路可以与其相连;其次，这些对象是无用的，即程序以后不会再使用这些对象。如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却占用内存。

## 使用Double brace初始化HashMap导致内存泄漏的风险

```
public class ReallyHeavyObject {
 
    // Just to illustrate...
    private int[] tonsOfValues;
    private Resource[] tonsOfResources;
 
    // This method almost does nothing
    public Map quickHarmlessMethod() {
        Map source = new HashMap(){{
            put("firstName", "John");
            put("lastName", "Smith");
            put("organizations", new HashMap(){{
                put("0", new HashMap(){{
                    put("id", "1234");
                }});
                put("abc", new HashMap(){{
                    put("id", "5678");
                }});
            }});
        }};
        return source;
        // Some more code here
    }
} 
```
当调用quickHarmlessMethod往外部暴露数据时，会导致ReallyHeavyObject这个大对象不能被垃圾回收器回收。因为Map里的每个匿名内部类都持有ReallyHeavyObject的引用。

- https://blog.jooq.org/2014/12/08/dont-be-clever-the-double-curly-braces-anti-pattern/

## 参考
- https://reinhard.codes/2016/07/30/double-brace-initialisation-and-java-initialisation-blocks/
- https://www.zhihu.com/question/36366039
- https://richardcao.me/2017/05/15/Java-Hashmap-Serializable/
- https://stackoverflow.com/questions/32790025/hashmap-not-serializable
- https://docs.oracle.com/javase/tutorial/java/javaOO/initial.html