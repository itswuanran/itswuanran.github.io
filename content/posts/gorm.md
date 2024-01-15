---
title: Gorm gen支持自定义查询语句
date: 2023-09-15 14:17:43
tags: 
- "Gorm"
---

## 背景


在使用[GORM Gen](https://github.com/go-gorm/gen) 作为ORM来写业务逻辑时，会有一些不那么如意的地方。

MySQL在5.7版本后就支持了json类型的存储，这为某个字段提供了一种结构化的能力。在部分后台场景会有针对json数据查询的场景，调研了下，可以使用json内置了部分函数`JSON_EXTRACT`, `JSON_OVERLAPS`, `JSON_CONTAINS`来支持。

Gorm针对这类场景有一个[datatypes](https://github.com/go-gorm/datatypes)项目，来支持这种结构化的查询条件。但接口设计和定义上还是比较鸡肋的，对json支持的也不够完整。

一旦涉及到稍复杂的SQL还继续使用Gen就比较吃力了，但本人还不想放弃，于是就去翻了底层源码。还是找到了一个增强的方法

## 详细设计

在Gorm接口设计上，可以看到Where()条件接收的是一个Condition接口

```go
type (
    // Condition query condition
    // field.Expr and subquery are expect value
    Condition interface {
        BeCond() interface{}
        CondError() error
    }
)
```
就很自然的想到了使用自定义Condition实现，代码也非常精简，如下：

```go
package cdatatypes

import (
    "gorm.io/gen/field"
    "gorm.io/gorm/clause"
)

type ExprCond struct {
    clause.Expr
    field.String
}

func Cond(expr clause.Expr) *ExprCond {
    return &ExprCond{
        Expr:   expr,
        String: field.String{},
    }
}

func (c *ExprCond) BeCond() interface{} { return c.Expr }

func (c *ExprCond) CondError() error { return nil }
```

在实际使用时，就可以通过以下的方式生成Where需要的条件。

```go
cdatatypes.Cond(gorm.Expr("JSON_OVERLAPS(JSON_EXTRACT(`query`, '$.query_list'), ?)", querys))

// JSON_OVERLAPS(JSON_EXTRACT(`query`, '$.query_list'), '["自定义查询"]')

```

这样既不会打破gorm gen提供的语义化模型，还很灵活的支持了各种自定义SQL。这样又可以开心的Coding了。