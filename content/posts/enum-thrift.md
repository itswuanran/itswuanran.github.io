---
title: Thrift中enum的一些探究
date: 2018-06-27 10:45:20
tags: 
- thrift
- rpc
categories:
- rpc
---
## 问题
在用注解定义的Thrift enum 中，如果客户端端和服务端的enum定义不同，比如调换了enum中的枚举值的顺序，就会发生调用端发送的枚举参数与服务端解析得到的枚举参数不一致的问题。

## 猜想
java 中的enum类的每一个具体的枚举值都有一个ordinal，代表其声明顺序，从零开始。thrift在序列化和反序列化时会将枚举值转换为一个整数传递，所以枚举值的具体含义与调用端和服务端各自的enum代码中声明顺序有关。

Thrift 注解的实现中对于thrift中每个关键字都有对应的编解码器，enum对应的为EnumThriftCodec<T extends Enum<T>，这个类继承了接口ThriftCodec<T> 。具体代码如下：

```java
package com.facebook.swift.codec.internal;

import com.facebook.swift.codec.ThriftCodec;
import com.facebook.swift.codec.metadata.ThriftEnumMetadata;
import com.facebook.swift.codec.metadata.ThriftType;
import com.google.common.base.Preconditions;
import org.apache.thrift.protocol.TProtocol;

import javax.annotation.concurrent.Immutable;
// 在类开始部分的注释就说明了EnumThriftCodec将enum编码成thrift里的i32，也就是int。枚举值会被编码成一个整数。

/**
 * EnumThriftCodec is a codec for Java enum types.  An enum is encoded as an I32 in Thrift, and this
 * class handles converting this vale to a Java enum constant.
 */
@Immutable
public class EnumThriftCodec<T extends Enum<T>> implements ThriftCodec<T>
{
    private final ThriftType type;
    private final ThriftEnumMetadata<T> enumMetadata;

    public EnumThriftCodec(ThriftType type)
    {
        this.type = type;
        enumMetadata = (ThriftEnumMetadata<T>) type.getEnumMetadata();
    }

    @Override
    public ThriftType getType()
    {
        return type;
    }

    @Override
    public T read(TProtocol protocol)
            throws Exception
    {
        int enumValue = protocol.readI32();
        if (enumValue >= 0) {
            // 检查当前解码的枚举类是否有显式声明的对应整数值，如果有，则直接根据声明的对应关系进行解码，如果没有显式声明对应关系，则直接获取当前枚举类下声明的所有枚举值，按照enumValue进行索引
            if (enumMetadata.hasExplicitThriftValue()) {
                T enumConstant = enumMetadata.getByEnumValue().get(enumValue);
                if (enumConstant != null) {
                    return enumConstant;
                }
            }
            else {
                T[] enumConstants = enumMetadata.getEnumClass().getEnumConstants();
                if (enumValue < enumConstants.length) {
                    return enumConstants[enumValue];
                }
            }
        }
        // unknown, throw unknown value exception
        throw new UnknownEnumValueException(
                String.format(
                        "Enum %s does not have a value for %s",
                        enumMetadata.getEnumClass(),
                        enumValue
                )
        );
    }

    @Override
    public void write(T enumConstant, TProtocol protocol)
            throws Exception
    {
        Preconditions.checkNotNull(enumConstant, "enumConstant is null");

        int enumValue;
        // 编码过程与解码过程基本一致，首先判断枚举类有没有显式声明的对应整数值，如果有则根据声明的对应关系进行编码，否则就直接按照其ordinal编码。ordinal()方法的具体实现就是返回枚举值的声明顺序的索引（从0开始）

        if (enumMetadata.hasExplicitThriftValue()) {
            enumValue = enumMetadata.getByEnumConstant().get(enumConstant);
        }
        else {
            enumValue = enumConstant.ordinal();
        }
        protocol.writeI32(enumValue);
    }
}

/**
 * Returns the ordinal of this enumeration constant (its position
 * in its enum declaration, where the initial constant is assigned
 * an ordinal of zero).
 *
 * Most programmers will have no use for this method.  It is
 * designed for use by sophisticated enum-based data structures, such
 * as {@link java.util.EnumSet} and {@link java.util.EnumMap}.
 *
 * @return the ordinal of this enumeration constant
 */

```

## 初步结论
现在总结一下，Thrift 注解方式会将枚举值编码成一个int进行网络传输，而在处理具体的枚举值与整数之间的对应关系的时候有两种策略：

如果枚举类显式声明了枚举值与整数之间的对应关系，则根据声明的规则进行编解码

如果枚举类中没有显式声明对应关系，则根据声明顺序的索引进行编解码。

具备了以上知识，就能解答文章最开始的问题了：如果客户端和服务端的枚举类里没有显式声明枚举值和整数值的对应关系，那么在编解码的时候的对应关系就是枚举值的声明顺序，如果两端的枚举类中枚举值的顺序不一致，就会导致两端编解码的的枚举值不一致。

## 深入探究
接下来再进一步探索，EnumThriftCodec如何判断一个枚举类是否声明了枚举值到整数的对应关系？为了回答这个问题，需要首先弄清楚EnumThriftCodec中的成员变量EnumMetadata的具体内容：

ThriftEnumMetadata.java
```java
/*
 * Copyright (C) 2012 Facebook, Inc.
 *
 * Licensed under the Apache License, Version 2.0 (the "License"); you may
 * not use this file except in compliance with the License. You may obtain
 * a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations
 * under the License.
 */
package com.facebook.swift.codec.metadata;
​
import com.facebook.swift.codec.ThriftEnumValue;
import com.google.common.base.Preconditions;
import com.google.common.collect.ImmutableList;
import com.google.common.collect.ImmutableMap;
​
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.Map;
​
import javax.annotation.concurrent.Immutable;
​
import static java.lang.String.format;
​
@Immutable
public class ThriftEnumMetadata<T extends Enum<T>>
{
    private final Class<T> enumClass;
    private final Map<Integer, T> byEnumValue;
    private final Map<T, Integer> byEnumConstant;
    private final String enumName;
    private final ImmutableList<String> documentation;
    private final ImmutableMap<T, ImmutableList<String>> elementDocs;
​
    public ThriftEnumMetadata(
            String enumName,
            Class<T> enumClass)
            throws RuntimeException
    {
        //构造函数
    }
​
    public String getEnumName()
    {
        return enumName;
    }
​
    public Class<T> getEnumClass()
    {
        return enumClass;
    }
​
    public boolean hasExplicitThriftValue()
    {
        return byEnumValue != null;
    }
​
    public Map<Integer, T> getByEnumValue()
    {
        return byEnumValue;
    }
​
    public Map<T, Integer> getByEnumConstant()
    {
        return byEnumConstant;
    }
​
    public ImmutableList<String> getDocumentation()
    {
        return documentation;
    }
​
    public Map<T, ImmutableList<String>> getElementsDocumentation()
    {
        return elementDocs;
    }
​
    @Override
    public boolean equals(Object o)
    {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
​
        final ThriftEnumMetadata<?> that = (ThriftEnumMetadata<?>) o;
​
        if (!enumClass.equals(that.enumClass)) {
            return false;
        }
​
        return true;
    }
​
    @Override
    public int hashCode()
    {
        return enumClass.hashCode();
    }
​
    @Override
    public String toString()
    {
        final StringBuilder sb = new StringBuilder();
        sb.append("ThriftEnumMetadata");
        sb.append("{enumClass=").append(enumClass);
        sb.append(", byThriftValue=").append(byEnumValue);
        sb.append('}');
        return sb.toString();
    }
}
​```
顾名思义，ThriftEnumMetadata表示一个thrift enum类的元数据，它的主要逻辑都集中在构造函数ThriftEnumMetadata( String enumName, Class<T> enumClass)中，其实现比较长，我们分两个部分来看：

```java
ThriftEnumMetadata(String enumName,  Class<T> enumClass) throws RuntimeException part 1

Method enumValueMethod = null;
for (Method method : enumClass.getMethods()) {
    if (method.isAnnotationPresent(ThriftEnumValue.class)) {
        Preconditions.checkArgument(
                Modifier.isPublic(method.getModifiers()),
                "Enum class %s @ThriftEnumValue method is not public: %s",
                enumClass.getName(),
                method);
        Preconditions.checkArgument(
                !Modifier.isStatic(method.getModifiers()),
                "Enum class %s @ThriftEnumValue method is static: %s",
                enumClass.getName(),
                method);
        Preconditions.checkArgument(
                method.getTypeParameters().length == 0,
                "Enum class %s @ThriftEnumValue method has parameters: %s",
                enumClass.getName(),
                method);
        Class<?> returnType = method.getReturnType();
        Preconditions.checkArgument(
                returnType == int.class || returnType == Integer.class,
                "Enum class %s @ThriftEnumValue method does not return int or Integer: %s",
                enumClass.getName(),
                method);
        enumValueMethod = method;
    }
}
```
第一部分的实现比较简单，首先给成员变量enumName和enumClass赋值，然后声明了一个Method类型的变量enumValueMethod，从代码逻辑我们可以看到这个enumValueMethod的几个特征：

包含注解@ThriftEnumValue

public方法

非static方法

参数列表为空，即不要求传入参数

返回值为int或Integer
 
接下来看构造函数的第二个部分：
```java
ThriftEnumMetadata(String enumName,  Class<T> enumClass) throws RuntimeException part 2

ImmutableMap.Builder<T, ImmutableList<String>> elementDocs = ImmutableMap.builder();
if (enumValueMethod != null) {
    ImmutableMap.Builder<Integer, T> byEnumValue = ImmutableMap.builder();
    ImmutableMap.Builder<T, Integer> byEnumConstant = ImmutableMap.builder();
    for (T enumConstant : enumClass.getEnumConstants()) {
        Integer value;
        try {
            value = (Integer) enumValueMethod.invoke(enumConstant);
        }
        catch (Exception e) {
            throw new RuntimeException(format("Enum class %s element %s get value method threw an exception", enumClass.getName(), enumConstant), e);
        }
        Preconditions.checkArgument(
                value != null,
                "Enum class %s element %s returned null for enum value: %s",
                enumClass.getName(),
                enumConstant
        );
​
        byEnumValue.put(value, enumConstant);
        byEnumConstant.put(enumConstant, value);
        elementDocs.put(enumConstant, ThriftCatalog.getThriftDocumentation(enumConstant));
    }
    this.byEnumValue = byEnumValue.build();
    this.byEnumConstant = byEnumConstant.build();
}
else {
    byEnumValue = null;
    byEnumConstant = null;
    for (T enumConstant : enumClass.getEnumConstants()) {
        elementDocs.put(enumConstant, ThriftCatalog.getThriftDocumentation(enumConstant));
    }
}
this.elementDocs = elementDocs.build();
this.documentation = ThriftCatalog.getThriftDocumentation(enumClass);
```
第二部分的主要逻辑有两个：
1）如果enumValueMethod不为空，则对当前枚举类下声明的每个枚举值调用enumValueMethod方法，并用返回结果填充两个map：byEnumValue和byEnumConstant，分别表示整数到枚举值和枚举值到整数的映射关系；
2）获取并保存枚举类声明的documention。

最后再看EnumThriftCodec是如何判断枚举类是否显式声明了枚举值与整数之间的对应关系的，即hasExplicitThriftValue方法的实现：
```java
hasExplicitThriftValue

public boolean hasExplicitThriftValue()
{
    return byEnumValue != null;
}
```
很简单，就是判断保存整数到枚举值的对应关系的byEnumValue是否为空。

## 最佳实践
Thrift 中注解开发时 enum 相关的几点建议

尽量保持调用端和服务端的 thrift 定义一致

在枚举类中定义@ThriftEnumValue方法来显式声明枚举值与整数的对应关系，避免使用默认的编解码规则

如果声明了带有@ThriftEnumValue的返回整数类型的无参public函数，请确保每个枚举值调用该方法的返回值都不一样（参考Object的hashcode方法）

举例
```java 

欠佳的例子：

欠佳的例子
/**
 * 没有提供枚举值到整数的对应关系，在编解码时会按照声明顺序进行索引
 */
@ThriftEnum
public enum ThriftAnnotatedEnum {
    FIRST_VALUE("first"),
    SECOND_VALUE("second");
​
    private String description;
​
    ThriftAnnotatedEnum(String description) {
        this.description = description;
    }
} 
建议的实现：

建议的实现 1

@ThriftEnum
public enum ThriftAnnotatedEnum {
    FIRST_VALUE("fist"),
    SECOND_VALUE("second");
​
    private String description;
​
    ThriftAnnotatedEnum(String description) {
        this.description = description;
    }
  
  //提供了返回int类型的无参public函数，建立从枚举值到整数的映射
    @ThriftEnumValue
    public int getIntValue() {
        return this.description.hashCode();
    }
}
建议的实现 2

@ThriftEnum
public enum ThriftAnnotatedEnum {
    FIRST_VALUE("fist", 0),
    SECOND_VALUE("second", 1);
​
    private String description;
    private int intValue;//直接在枚举类定义整数类型的成员变量用于标识
​
    ThriftAnnotatedEnum(String description, int intValue) {
        this.description = description;
        this.intValue = intValue;
    }
​
    @ThriftEnumValue
    public int getIntValue() {
        return intValue;
    }
}
补充
IDL方式中的enum的实现细节

TweetType.thrift

enum TweetType {
    TWEET,
    RETWEET = 2,
    DM = 0xa,
    REPLY
}
TweetType.java

public enum TweetType implements TEnum {
  TWEET(0),
  RETWEET(2),
  DM(10),
  REPLY(11);
​
  private final int value;
​
  private TweetType(int value) {
    this.value = value;
  }
​
  /**
   * Get the integer value of this enum value, as defined in the Thrift IDL.
   */
  public int getValue() {
    return value;
  }
​
  /**
   * Find a the enum type by its integer value, as defined in the Thrift IDL.
   * @return null if the value is not found.
   */
  public static TweetType findByValue(int value) { 
    switch (value) {
      case 0:
        return TWEET;
      case 2:
        return RETWEET;
      case 10:
        return DM;
      case 11:
        return REPLY;
      default:
        return null;
    }
  }
}
```
使用IDL文件编译生成的枚举类下有两个方法getValue和findByValue，用于定义枚举值到整数的映射
