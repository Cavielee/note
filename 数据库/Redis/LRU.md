缓存清理算法 LRU

插入元素 set()：

1. 首先判断缓存 Map 中是否有该 key ，有则删除
2. 插入新的 Entry