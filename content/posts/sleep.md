---
title: 你知道MySQL中的sleep的含义吗
date: 2023-02-08 18:19:38
tags: 
- sleep
categories:
- sleep
---

## select sleep
select sleep函数会在每个命中的行都执行休眠操作，也就是说，如果查询返回100行，SQL整体执行耗时就放大100倍

## SQL注入案例
- https://planet.mysql.com/entry/?id=5994442

## sleep导致alter表等待
As we see in the example, ALTER table will wait until it can get a lock on post table, and this blocks every other select from now on to the table.
