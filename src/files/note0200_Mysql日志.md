
###  通过一条更新语句 把 WAL（write ahead logging）机制 ，rodolog 和 binglog 两个日志 和 两阶段提交串 起来  

> WAL机制 在 spakrstreaming  flink 中的容错机制中也有用到，看起来 WAL 机制是一个通用的机制 
  
> 两阶段提交机制在 多线程 和  分布式系统设计中 对于 异步事务的 提交 也用到过  看起来 来那个阶段提交 也是一个通用的思想    


```
第一步：  
学习并且想清楚 这些 机制的逻辑 之后 
列出些机制的实现步骤 流程图 或者伪代码 

第二步：
参看源码：验证你的逻辑是否正确
 
```  

## worklist

1 rodolog 什么时候执行 怎么执行

2 binglog 什么时候执行 怎么执行  

3 两阶段提交 怎么执行的 反证法证明如果不来那个阶段提交会有什么问题 会不会数据不一致   


# rodolog 什么时候执行 怎么执行 

## 追踪update语句的执行流程  

在0201中跟踪更新流程的时候还是没有找到 redolog 写入的过程   

binglog 的过程也没有找到 

又发现了一个 undolog

各种buffer  

各种io 

各种file 

各种算法 

各种锁  事务 


# 插入流程分析  







