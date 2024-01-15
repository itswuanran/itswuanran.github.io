---
title: 理解RESTful中的PUT请求
date: 2018-01-15 11:07:18
---

## RESTful中的PUT和POST

According to the HTTP/1.1 Spec:
```
The POST method is used to request that the origin server accept the entity enclosed in the request as a new subordinate of the resource identified by the Request-URI in the Request-Line
```
In other words, POST is used to create.
```
The PUT method requests that the enclosed entity be stored under the supplied Request-URI. If the Request-URI refers to an already existing resource, the enclosed entity SHOULD be considered as a modified version of the one residing on the origin server. If the Request-URI does not point to an existing resource, and that URI is capable of being defined as a new resource by the requesting user agent, the origin server can create the resource with that URI."
```
That is, PUT is used to create or update.

So, which one should be used to create a resource? Or one needs to support both?

Better is to choose between PUT and POST based on idempotence of the action.

PUT implies putting a resource - completely replacing whatever is available at the given URL with a different thing. By definition, a PUT is idempotent. Do it as many times as you like, and the result is the same. x=5 is idempotent. You can PUT a resource whether it previously exists, or not (eg, to Create, or to Update)!

POST updates a resource, adds a subsidiary resource, or causes a change. A POST is not idempotent, in the way that x++ is not idempotent.

By this argument, PUT is for creating when you know the URL of the thing you will create. POST can be used to create when you know the URL of the "factory" or manager for the category of things you want to create.

so:

POST /expense-report
or:

PUT  /expense-report/10929

> 总结后一句话：PUT操作应该保持幂等，PUT即可新增又可以修改，POST每次请求都为新增。

## 参考
- https://stackoverflow.com/questions/630453/put-vs-post-in-rest