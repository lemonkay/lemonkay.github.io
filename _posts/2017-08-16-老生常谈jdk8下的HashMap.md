---
　　layout: default
　　title: 我的Blog
---
### 老生常谈jdk8下的HashMap
1. > [美团的一篇文章](https://tech.meituan.com/java_hashmap.html)


2. 红黑树是满足下面三个性质的二叉搜索树：
    - 节点有红，黑的属性，根是黑节点
    - 不能出现连续的红节点；
    - 完美黑色平衡，即任意空链接到根节点的路径上的黑节点数量相同。



3. resize 并发下出现环链表的情况，get()方法，e.next会出现无限循环

4. 针对HashMap线程不安全的问题，并发包下的ConcurrentHashMap可以解决这个问题：
    > [java8下实现](https://yq.aliyun.com/articles/36781)