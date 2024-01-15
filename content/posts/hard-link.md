---
title: 硬链接原理秒删大文件
date: 2018-05-08 15:46:49
tags:
- hardlink
-categories:
- System
---
## 硬链接基础
当多个文件共同指向同一inode、inode链接数N>1、删除任何一个文件都是巨快
因为、此时删除的仅仅是指向inode的指针，而当N=1时、则不一样了、此时删除的文件相关的所有数据块、所以慢。

```bash
ln test.ibd test.ibd.hlk
```

## 参考
- https://www.ibm.com/developerworks/cn/linux/l-cn-hardandsymb-links/