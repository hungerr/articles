## Redis SCAN命令实现有限保证的原理

SCAN命令可以为用户保证：从完整遍历开始直到完整遍历结束期间，一直存在于数据集内的所有元素都会被完整遍历返回，但是同一个元素可能会被返回多次。如果一个元素是在迭代过程中被添加到数据集的，又或者是在迭代过程中从数据集中被删除的，那么这个元素可能会被返回，也可能不会返回。

这是如何实现的呢，先从Redis中的字典dict开始，Redis的数据库是使用dict作为底层实现的。

### 字典数据类型

Redis中的字典由`dict.h/dict`结构表示：

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;

typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
``` 

字典由两个哈希表`dictht`构成，主要用做rehash，平常主要使用ht[0]哈希表。

哈希表由一个成员为`dictEntry`的数组构成，`size`属性记录了数组的大小，`used`属性记录了已有节点的数量，`sizemask`属性的值等于`size - 1`。数组大小一般是2<sup>n</sup>，所以`sizemask`二进制是`0b11111...`，主要用作掩码，和哈希值一起决定key应该放在数组的哪个位置。

求key在数组中的索引的计算方法如下：

```c
index = hash & d->ht[table].sizemask; 
```

也就是根据掩码求低位值。

### rehash的问题

字典rehash时会使用两个哈希表，首先为ht[1]分配空间，如果是扩展操作，ht[1]的大小为第一个大于等于2倍`ht[0].used`的2<sup>n</sup>，如果是收缩操作，ht[1]的大小为第一个大于等于`ht[0].used`的2<sup>n</sup>。然后将ht[0]的所有键值对rehash到ht[1]中，最后释放ht[0]，将ht[1]设置为ht[0]，新创建一个空白哈希表当做ht[1]。rehash不是一次完成的，而是分多次、渐进式地完成。

举个例子，现在将一个`size`为4的哈希表`ht[0]`(sizemask为11,index = hash & 0b11)rehash至一个`size`为8的哈希表`ht[1]`(sizemask为111,index = hash & 0b111)。

ht[0]中处于index=0位置的key低两位为`00`，那么rehash至ht[1]时index取低三位可能为`000(0)`和`100(4)`。也就是ht[0]中bucket0中的元素rehash之后分散于ht[1]的bucket0与bucket4，以此类推，对应关系为：

```
    ht[0]  ->  ht[1]
    ----------------
      0    ->   0,4 
      1    ->   1,5
      2    ->   2,6
      3    ->   3,7
```

如果SCAN命令采取0->1->2->3的顺序进行遍历，就会出现重复和遗漏的问题：



- 扩展操作中，如果返回游标0时正在进行rehash,ht[0]中的bucket0中的部分数据可能已经rehash到ht[1]中的bucket[0]或者bucket[4]，在ht[1]中从bucket1开始遍历，遍历至bucket4时，其中的元素已经在ht[0]中的bucket0中遍历过，这就了产生重复问题。
- aaaa







