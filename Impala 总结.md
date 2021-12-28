# Impala 总结

## 1、Impala高性能探秘之Runtime Filter

https://blog.csdn.net/yu616568/article/details/77073166

**JOIN节点会依赖于小表返回的每一条记录的等值ON条件中使用的列生成一个Bloom Filter信息，然后将Bloom Filter和大表中的等值ON条件使用的列生成一个过滤条件。这个条件就是Runtime Filte在Impala中的作用**

```
COMPUTE STATS 收集信息，优化性能。
```



