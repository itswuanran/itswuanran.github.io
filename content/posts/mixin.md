---
title: kotlin扩展
date: 2018-02-08 17:18:01
tags: 
- 语言设计
categories:
- 语言设计
---

## 什么是mixin?

In object-oriented programming languages, a Mixin is a class that contains methods for use by other classes without having to be the parent class of those other classes. How those other classes gain access to the mixin's methods depends on the language. Mixins are sometimes described as being "included" rather than "inherited".

Mixin即mix in，混入的意思。
和多重继承类似，但通常混入Mixin的类和Mixin类本身不是is-a的关系，混入Mixin类是为了添加某些（可选的）功能。

自由地混入Mixin类就可以灵活地为被混入的类添加不同的功能。

传统的接口并不包含实现，而Mixin包含实现。实际上Mixin的作用和Java中的众多以able结尾的接口很相似。不同的是Mixin提供了（默认）实现，而 Java 中实现了able接口的类需要类自身来实现这些混入的功能（Serializable 接口是个例外）。

为了解决多重继承的问题，Java引入了接口 （interface）技术，Lisp、Ruby引入了 Mix-in 技术。

mix-in 是一个行为的集合，而这个行为可以被加到任意class里，然而在一些情况下，使用mix-in的类，需要满足一些协议（contract）。和接口的不同点，有两个。1.  如果有协议（contract）的话，协议是被声明在mix-in的文件内的。也就是说，更容易复用。2.  接口，不包括如何实现。而mix-in是关于如何容易地重用实现的。

个人理解mixin目的是增强类，不管是通过多继承还是其他组合方式。

## kotlin扩展
kotlin扩展是一个非常重要的特性，它是用来扩展类型的能力

Kotlin 能够扩展一个类的新功能而无需继承该类或者使用像装饰者这样的设计模式。 这通过叫做 `扩展` 的特殊声明完成。 例如，你可以为一个你不能修改的、来自第三方库中的类编写一个新的函数。 这个新增的函数就像那个原始类本来就有的函数一样，可以用普通的方法调用。 这种机制称为 `扩展函数` 。此外，也有 `扩展属性` ， 允许你为一个已经存在的类添加新的属性。

- https://www.kotlincn.net/docs/reference/extensions.html

## 参考
- https://en.wikipedia.org/wiki/Mixin#
- https://www.zhihu.com/question/20778853
