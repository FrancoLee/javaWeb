## master

---

[参考1(其实就是搬运)](https://www.cnblogs.com/zziawanblog/p/6577383.html)

[参考2(其实就是搬运)](https://blog.csdn.net/xiaoyu_BD/article/details/82016395)

`master`作用：

+ 集群层面的设置
+ 集群内节点信息
+ 各索引的设置，映射，分析器和别名等
+ 索引内各分片所在的节点位置

每个节点都会保存以上信息，当只有`master`节点能修改这些信息。

选举算法：



​	每个节点都有一个独特的`id`。每次选举`master`的时候，节点会通过`ping方法(非命令行的ping)`去发现连接着的活跃节点。将