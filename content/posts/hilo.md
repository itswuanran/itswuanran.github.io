---
title: hilo算法
date: 2018-06-11 10:32:47
tags: 
- Hibernate
- hilo
categories:
- MySQL
---

## 介绍
hilo算法是一个主键生成策略。
在算法中id的计算公式是：
```
hi * (max_lo + 1) + lo
```
代码如下：
```java

private class HiloOptimizer {

    private String prefix;
    private int maxLo;
    private int lo;
    private long hi;
    private long lastValue;

    public HiloOptimizer(String prefix, int maxLo) {
        this.prefix = prefix != null ? prefix.replace("{", "${") : "";
        this.maxLo = maxLo;            //最大低位  
        this.lo = maxLo + 1;           //最低初始位  
    }

    public synchronized String generate() {
        if (lo > maxLo) {                    //当低位超过最大高位  
            lastValue = getLastValue();  //表示hi位的进位次数，从数据库id管理表获取  
            lo = lastValue == 0 ? 1 : 0;     //低位归0  
            hi = lastValue * (maxLo + 1);    //高位进位  
        }
        return String.valueOf(hi + lo++);    //低位最后自增lo++  
    }
} 
```

## 分析
hilo算法的优势：
解决了一个效率问题：
每次取id号都要从数据库查一遍。
分析了hilo算法，它其实优化了类似oracle序列这种由数据库管理的id，每次插入都必须查询一遍数据库这种模式，用一个静态变量低位当成一个oracle序列，当它增长超过设置的最大低位时再查询数据库，减少查询数据库的次数maxLo倍！从而提升了项目的效率和性能。

## 自增表数据删除
业务时常会依赖表主键自增id来生成订单号，但我们一般只需第一条，历史数据都是可删除的。
通过重命名表可达到这个效果：
```sql
RENAME TABLE `tt` TO `tt_old`, `tt_new` TO `tt`;
```