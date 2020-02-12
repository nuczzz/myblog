## k8s client informer机制源码分析

### 什么是informer？
informer是一个带有本地缓存和索引机制的可以注册event handler的client，有这么几个概念：

本地缓存：本地缓存在源码中叫做store，是用来缓存相关资源数据的
索引：可根据索引从informer中过滤查找相关资源
event handler：注册事件处理函数

//todo
