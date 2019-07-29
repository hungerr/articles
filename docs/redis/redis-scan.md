## Redis SCAN命令实现有限保证的原理

SCAN命令可以为用户保证：从完整遍历开始直到完整遍历结束期间，一直存在于数据集内的所有元素都会被完整遍历返回，但是同一个元素可能会被返回多次。如果一个元素是在迭代过程中被添加到数据集的，又或者是在迭代过程中从数据集中被删除的，那么这个元素可能会被返回，也可能不会返回。

这是如何实现的呢，先从Redis中的字典dict开始。Redis的数据库是使用dict作为底层实现的。

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

举个例子，现在将一个size为4的哈希表`ht[0]`(sizemask为11, index = hash & 0b11)rehash至一个size为8的哈希表`ht[1]`(sizemask为111, index = hash & 0b111)。

ht[0]中处于bucket0位置的key的哈希值低两位为`00`，那么rehash至ht[1]时index取低三位可能为`000(0)`和`100(4)`。也就是ht[0]中bucket0中的元素rehash之后分散于ht[1]的bucket0与bucket4，以此类推，对应关系为：

```
    ht[0]  ->  ht[1]
    ----------------
      0    ->   0,4 
      1    ->   1,5
      2    ->   2,6
      3    ->   3,7
```

如果SCAN命令采取0->1->2->3的顺序进行遍历，就会出现如下问题：

- 扩展操作中，如果返回游标1时正在进行rehash，ht[0]中的bucket0中的部分数据可能已经rehash到ht[1]中的bucket[0]或者bucket[4]，在ht[1]中从bucket1开始遍历，遍历至bucket4时，其中的元素已经在ht[0]中的bucket0中遍历过，这就产生了重复问题。
- 缩小操作中，当返回游标5，但缩小后哈希表的size只有4，如何重置游标？

### SCAN的遍历顺序

`SCAN`命令的遍历顺序，可以举一个例子看一下：

```bash
127.0.0.1:6379[3]> keys *
1) "bar"
2) "qux"
3) "baz"
4) "foo"
127.0.0.1:6379[3]> scan 0 count 1
1) "2"
2) 1) "bar"
127.0.0.1:6379[3]> scan 2 count 1
1) "1"
2) 1) "foo"
127.0.0.1:6379[3]> scan 1 count 1
1) "3"
2) 1) "qux"
   2) "baz"
127.0.0.1:6379[3]> scan 3 count 1
1) "0"
2) (empty list or set)
```

可以看出顺序是`0->2->1->3`，很难看出规律，转换成二进制观察一下：

```
00 -> 10 -> 01 -> 11
```

二进制就很明了了，遍历采用的顺序也是加法，但每次是高位加1的，也就是从左往右相加、从高到低进位的。

### SCAN源码

`SCAN`遍历字典的源码在`dict.c/dictScan`，分两种情况，字典不在进行rehash或者正在进行rehash。

不在进行rehash时，游标是这样计算的：

```c
m0 = t0->sizemask;

// 将游标的umask位的bit都置为1
v |= ~m0;

// 反转游标
v = rev(v);
// 反转后+1，达到高位加1的效果
v++;
// 再次反转复位
v = rev(v);
```

当size为4时，sizemask为3(00000011)，游标计算过程：

```
         v |= ~m0    v = rev(v)    v++       v = rev(v)

00000000(0) -> 11111100 -> 00111111 -> 01000000 -> 00000010(2)

00000010(2) -> 11111110 -> 01111111 -> 10000000 -> 00000001(1)

00000001(1) -> 11111101 -> 10111111 -> 11000000 -> 00000011(3)

00000011(3) -> 11111111 -> 11111111 -> 00000000 -> 00000000(0)
```

遍历size为4时的游标状态转移为`0->2->1->3`。

同理,size为8时的游标状态转移为`0->4->2->6->1->5->3->7`，也就是`000->100->010->110->001->101->011->111`。

再结合前面的rehash：

```
    ht[0]  ->  ht[1]
    ----------------
      0    ->   0,4 
      1    ->   1,5
      2    ->   2,6
      3    ->   3,7
```

可以看出，当size由小变大时，所有原来的游标都能在大的哈希表中找到相应的位置，并且顺序一致，不会重复读取并且不会遗漏。

当size由大变小的情况，假设size由8变为了4，分两种情况，一种是游标为`0,2,1,3`中的一种，此时继续读取，也不会遗漏和重复。

但如果游标返回的不是这四种，例如返回了7，7&11之后变为了3，所以会从size为4的哈希表的bucket3开始继续遍历，而bucket3包含了size为8的哈希表中的bucket3与bucket7，所以会造成重复读取size为8的哈希表中的bucket3的情况。

所以，redis里rehash从小到大时，SCAN命令不会重复也不会遗漏。而从大到小时，有可能会造成重复但不会遗漏。

当正在进行rehash时，游标计算过程：

```c
        /* Make sure t0 is the smaller and t1 is the bigger table */
        if (t0->size > t1->size) {
            t0 = &d->ht[1];
            t1 = &d->ht[0];
        }

        m0 = t0->sizemask;
        m1 = t1->sizemask;

        /* Emit entries at cursor */
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        de = t0->table[v & m0];
        while (de) {
            next = de->next;
            fn(privdata, de);
            de = next;
        }

        /* Iterate over indices in larger table that are the expansion
         * of the index pointed to by the cursor in the smaller table */
        do {
            /* Emit entries at cursor */
            if (bucketfn) bucketfn(privdata, &t1->table[v & m1]);
            de = t1->table[v & m1];
            while (de) {
                next = de->next;
                fn(privdata, de);
                de = next;
            }

            /* Increment the reverse cursor not covered by the smaller mask.*/
            v |= ~m1;
            v = rev(v);
            v++;
            v = rev(v);

            /* Continue while bits covered by mask difference is non-zero */
        } while (v & (m0 ^ m1));
```

算法会保证t0是较小的哈希表，不是的话t0与t1互换，先遍历t0中游标所在的bucket，然后再遍历较大的t1。

求下一个游标的过程基本相同，只是把`m0`换成了rehash之后的哈希表的`m1`，同时还加了一个判断条件:

```
v & (m0 ^ m1)
```

size4的m0为`00000011`，size8的m1为`00000111`，`m0 ^ m1`取值为`00000100`，即取二者mask的不同位，看游标在这些标志位是否为1。

假设游标返回了2，并且正在进行rehash，此时size由4变成了8，二者mask的不同位是低第三位。

首先遍历t0中的bucket2，然后遍历t1中的bucket2，公式计算出的下一个游标为6(00000110)，低第三位为1，继续循环，遍历t1中的bucket6，然后计算游标为1，结束循环。

所以正在rehash时，是两个哈希表都遍历的，以避免遗漏的情况。