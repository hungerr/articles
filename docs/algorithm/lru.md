# LRU算法解析

**LRU**是`Least Recently Used`的缩写，即最近最少使用，常用于页面置换算法，是为虚拟页式存储管理服务的。

现代操作系统提供了一种对主存的抽象概念`虚拟内存`，来对主存进行更好地管理。他将主存看成是一个存储在磁盘上的地址空间的高速缓存，在主存中只保存活动区域，并根据需要在主存和磁盘之间来回传送数据。`虚拟内存`被组织为存放在磁盘上的N个连续的字节组成的数组，每个字节都有唯一的虚拟地址，作为到数组的索引。`虚拟内存`被分割为大小固定的数据块`虚拟页(Virtual Page,VP)`，这些数据块作为主存和磁盘之间的传输单元。类似地，物理内存被分割为`物理页(Physical Page,PP)`。

`虚拟内存`使用`页表`来记录和判断一个`虚拟页`是否缓存在物理内存中：

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/algorithm/lru-page-table.png)

如上图所示，当CPU访问`虚拟页VP3`时，发现VP3并未缓存在物理内存之中，这称之为`缺页`，现在需要将VP3从磁盘复制到物理内存中，但在此之前，为了保持原有空间的大小，需要在物理内存中选择一个`牺牲页`，将其复制到磁盘中，这称之为`交换`或者`页面调度`，图中的`牺牲页`为VP4。把哪个页面调出去可以达到调动尽量少的目的？最好是每次调换出的页面是所有内存页面中最迟将被使用的——这可以最大限度的推迟页面调换，这种算法，被称为理想页面置换算法，但这种算法很难完美达到。

为了尽量减少与理想算法的差距，产生了各种精妙的算法，`LRU`算法便是其中一个。

## LRU原理
> LRU 算法的设计原则是：如果一个数据在最近一段时间没有被访问到，那么在将来它被访问的可能性也很小。也就是说，当限定的空间已存满数据时，应当把最久没有被访问到的数据淘汰。

根据[LRU原理和Redis实现](https://zhuanlan.zhihu.com/p/34133067)所示，假定系统为某进程分配了3个物理块，进程运行时的页面走向为 7 0 1 2 0 3 0 4，开始时3个物理块均为空，那么`LRU`算法是如下工作的：

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/algorithm/lru-v-stack.jpg)

## 基于哈希表和双向链表的LRU算法实现

如果要自己实现一个`LRU`算法，可以用哈希表加双向链表实现：

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/algorithm/lru-hash-link.jpg)

设计思路是，使用哈希表存储 key，值为链表中的节点，节点中存储值，双向链表来记录节点的顺序，头部为最近访问节点。

`LRU`算法中有两种基本操作：

- `get(key)`：查询key对应的节点，如果key存在，将节点移动至链表头部。
- `set(key, value)`： 设置key对应的节点的值。如果key不存在，则新建节点，置于链表开头。如果链表长度超标，则将处于尾部的最后一个节点去掉。如果节点存在，更新节点的值，同时将节点置于链表头部。

## LRU缓存机制

`leetcode`上有一道关于[`LRU缓存机制`](https://leetcode-cn.com/problems/lru-cache/)的题目：
>运用你所掌握的数据结构，设计和实现一个  LRU (最近最少使用) 缓存机制。它应该支持以下操作： 获取数据 get 和 写入数据 put 。

>获取数据 get(key) - 如果密钥 (key) 存在于缓存中，则获取密钥的值（总是正数），否则返回 -1。
写入数据 put(key, value) - 如果密钥不存在，则写入其数据值。当缓存容量达到上限时，它应该在写入新数据之前删除最近最少使用的数据值，从而为新的数据值留出空间。

>**进阶:**

>你是否可以在 O(1) 时间复杂度内完成这两种操作？

>**示例:**
>```
>LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );
>
>cache.put(1, 1);
>cache.put(2, 2);
>cache.get(1);       // 返回  1
>cache.put(3, 3);    // 该操作会使得密钥 2 作废
>cache.get(2);       // 返回 -1 (未找到)
>cache.put(4, 4);    // 该操作会使得密钥 1 作废
>cache.get(1);       // 返回 -1 (未找到)
>cache.get(3);       // 返回  3
>cache.get(4);       // 返回  4
>```

我们可以自己实现双向链表，也可以使用现成的数据结构，`python`中的数据结构`OrderedDict`是一个有序哈希表，可以记住加入哈希表的键的顺序，相当于同时实现了哈希表与双向链表。`OrderedDict`是将最新数据放置于末尾的:

```python
In [35]: from collections import OrderedDict

In [36]: lru = OrderedDict()

In [37]: lru[1] = 1

In [38]: lru[2] = 2

In [39]: lru
Out[39]: OrderedDict([(1, 1), (2, 2)])

In [40]: lru.popitem()
Out[40]: (2, 2)
```

`OrderedDict`有两个重要方法：
- `popitem(last=True)`: 返回一个键值对，当last=True时，按照`LIFO`的顺序，否则按照`FIFO`的顺序。
- `move_to_end(key, last=True)`: 将现有 key 移动到有序字典的任一端。 如果 last 为True（默认）则将元素移至末尾；如果 last 为False则将元素移至开头。

删除数据时，可以使用`popitem(last=False)`将开头最近未访问的键值对删除。访问或者设置数据时，使用`move_to_end(key, last=True)`将键值对移动至末尾。

代码实现：
```python
from collections import OrderedDict


class LRUCache:
    def __init__(self, capacity: int):
        self.lru = OrderedDict()
        self.capacity = capacity
        
    def get(self, key: int) -> int:
        self._update(key)
        return self.lru.get(key, -1)
        

    def put(self, key: int, value: int) -> None:
        self._update(key)
        self.lru[key] = value
        if len(self.lru) > self.capacity:
            self.lru.popitem(False)
         
    def _update(self, key: int):
        if key in self.lru:
            self.lru.move_to_end(key)
```

## OrderedDict源码分析

`OrderedDict`其实也是用哈希表与双向链表实现的：
```python
class OrderedDict(dict):
    'Dictionary that remembers insertion order'
    # An inherited dict maps keys to values.
    # The inherited dict provides __getitem__, __len__, __contains__, and get.
    # The remaining methods are order-aware.
    # Big-O running times for all methods are the same as regular dictionaries.

    # The internal self.__map dict maps keys to links in a doubly linked list.
    # The circular doubly linked list starts and ends with a sentinel element.
    # The sentinel element never gets deleted (this simplifies the algorithm).
    # The sentinel is in self.__hardroot with a weakref proxy in self.__root.
    # The prev links are weakref proxies (to prevent circular references).
    # Individual links are kept alive by the hard reference in self.__map.
    # Those hard references disappear when a key is deleted from an OrderedDict.

    def __init__(*args, **kwds):
        '''Initialize an ordered dictionary.  The signature is the same as
        regular dictionaries.  Keyword argument order is preserved.
        '''
        if not args:
            raise TypeError("descriptor '__init__' of 'OrderedDict' object "
                            "needs an argument")
        self, *args = args
        if len(args) > 1:
            raise TypeError('expected at most 1 arguments, got %d' % len(args))
        try:
            self.__root
        except AttributeError:
            self.__hardroot = _Link()
            self.__root = root = _proxy(self.__hardroot)
            root.prev = root.next = root
            self.__map = {}
        self.__update(*args, **kwds)

    def __setitem__(self, key, value,
                    dict_setitem=dict.__setitem__, proxy=_proxy, Link=_Link):
        'od.__setitem__(i, y) <==> od[i]=y'
        # Setting a new item creates a new link at the end of the linked list,
        # and the inherited dictionary is updated with the new key/value pair.
        if key not in self:
            self.__map[key] = link = Link()
            root = self.__root
            last = root.prev
            link.prev, link.next, link.key = last, root, key
            last.next = link
            root.prev = proxy(link)
        dict_setitem(self, key, value)
```
由源码看出，`OrderedDict`使用`self.__map = {}`作为哈希表，其中保存了`key`与链表中的节点`Link()`的键值对，`self.__map[key] = link = Link()`:
```python
class _Link(object):
    __slots__ = 'prev', 'next', 'key', '__weakref__'
```
节点`Link()`中保存了指向前一个节点的指针`prev`，指向后一个节点的指针`next`以及`key`值。

而且，这里的链表是一个环形双向链表,`OrderedDict`使用一个哨兵元素`root`作为链表的**head**与**tail**：
```python
   self.__hardroot = _Link()
   self.__root = root = _proxy(self.__hardroot)
    root.prev = root.next = root
```
由`__setitem__`可知，向`OrderedDict`中添加新值时，链表变为如下的环形结构：
```
         next             next             next
   root <----> new node1 <----> new node2 <----> root
         prev             prev             prev
```
`root.next`为链表的第一个节点，`root.prev`为链表的最后一个节点。

由于`OrderedDict`继承自`dict`，键值对是保存在`OrderedDict`自身中的，链表节点中只保存了`key`，并未保存`value`。

如果我们要自己实现的话，无需如此复杂，可以将`value`置于节点之中，链表只需要实现插入最前端与移除最后端节点的功能即可：
```python
from _weakref import proxy as _proxy


class Node:
    __slots__ = ('prev', 'next', 'key', 'value', '__weakref__')


class LRUCache:

    def __init__(self, capacity: int):
        self.__hardroot = Node()
        self.__root = root = _proxy(self.__hardroot)
        root.prev = root.next = root
        self.__map = {}
        self.capacity = capacity
        
    def get(self, key: int) -> int:
        if key in self.__map:
            self.move_to_head(key)
            return self.__map[key].value
        else:
            return -1
         
    def put(self, key: int, value: int) -> None:
        if key in self.__map:
            node = self.__map[key]
            node.value = value
            self.move_to_head(key)
        else:
            node = Node()
            node.key = key
            node.value = value
            self.__map[key] = node
            self.add_head(node)
            if len(self.__map) > self.capacity:
                self.rm_tail()
        
    def move_to_head(self, key: int) -> None:
        if key in self.__map:
            node = self.__map[key]
            node.prev.next = node.next
            node.next.prev = node.prev
            head = self.__root.next
            self.__root.next = node
            node.prev = self.__root
            node.next = head
            head.prev = node
    
    def add_head(self, node: Node) -> None:
        head = self.__root.next
        self.__root.next = node
        node.prev = self.__root
        node.next = head
        head.prev = node
    
    def rm_tail(self) -> None:
        tail = self.__root.prev
        del self.__map[tail.key]
        tail.prev.next = self.__root
        self.__root.prev = tail.prev
```

## node-lru-cache
在实际应用中，要实现`LRU`缓存算法，还要实现很多额外的功能。

有一个用`javascript`实现的很好的[node-lru-cache](https://github.com/isaacs/node-lru-cache)包：
```javascript
var LRU = require("lru-cache")
  , options = { max: 500
              , length: function (n, key) { return n * 2 + key.length }
              , dispose: function (key, n) { n.close() }
              , maxAge: 1000 * 60 * 60 }
  , cache = new LRU(options)
  , otherCache = new LRU(50) // sets just the max size

cache.set("key", "value")
cache.get("key") // "value"
```
这个包不是用缓存`key`的数量来判断是否要启动`LRU`淘汰算法，而是使用保存的键值对的实际大小来判断。选项`options`中可以设置缓存所占空间的上限`max`，判断键值对所占空间的函数`length`，还可以设置键值对的过期时间`maxAge`等，有兴趣的可以看下。

### 参考链接
- [LRU原理和Redis实现——一个今日头条的面试题](https://zhuanlan.zhihu.com/p/34133067)