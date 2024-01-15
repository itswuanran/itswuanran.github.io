---
title: Gorm gen支持自定义查询语句
date: 2023-08-15 14:17:43
tags: 
- "Gorm"
---

## 背景



gorm针对json字段有个专门的datatypes包，但对json支持的不够完整。一旦涉及到稍复杂的SQL就无能为力。
查了下gorm的官方文档，发现在gorm gen模式下并没有提供很优雅的方式来实现自定义SQL语句查询条件。

于是想给这部分能力做一次增强，在接口设计上，可以看到Where()条件接收的是一个Condition接口
```
type (
    // Condition query condition
    // field.Expr and subquery are expect value
    Condition interface {
        BeCond() interface{}
        CondError() error
    }
)
```

因此想到了使用自定义Condition实现，代码也非常精简，如下：

```
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

在实际使用时，通过以下的语句生成Condition接口

```
cdatatypes.Cond(gorm.Expr("JSON_OVERLAPS(JSON_EXTRACT(`query`, '$.query_list'), ?)", querys))

// JSON_OVERLAPS(JSON_EXTRACT(`query`, '$.query_list'), '["自定义查询"]')

```