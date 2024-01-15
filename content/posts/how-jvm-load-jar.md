---
title: JVM加载jar包的顺序
date: 2018-08-23 22:34:15
---
## JVM加载jar的顺序
同一个目录下，jvm加载jar包顺序是无法保证的，每个系统的都不一样，甚至同一个系统不同的时刻加载都不一样。良好设计的系统不应该依赖详细的顺序。
```
The order in which the JAR files in a directory are enumerated in the expanded class path is not specified and may vary from platform to platform and even from moment to moment on the same machine. A well-constructed application should not depend upon any particular order. If a specific order is required, then the JAR files can be enumerated explicitly in the class path.
```
jvm在不同机器上加载jar包顺序与文件系统存在关系，jvm在linux通过jar文件在文件系统中生成顺序来进行加载，在linux底层是inode数据机构，是用inode来指示文件的。

## 参考

- https://www.cnblogs.com/saaav/p/7716179.html
- https://mp.weixin.qq.com/s?__biz=MzIxNDQ5NzI4OA==&mid=2247484971&idx=1&sn=954448152b5f0f632662b456509ce0f7
- https://www.zhihu.com/question/56147205/answer/220271109