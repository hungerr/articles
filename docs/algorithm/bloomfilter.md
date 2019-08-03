## 浅谈布隆过滤器Bloom Filter

先从一道面试题开始：

> 给A,B两个文件，各存放50亿条URL，每条URL占用64字节，内存限制是4G，让你找出A,B文件共同的URL。

这个问题的本质在于判断一个元素是否在一个集合中。哈希表以O(1)的时间复杂度来查询元素，但付出了空间的代价。在这个大数据问题中，就算哈希表有100%的空间利用率，也至少需要50亿*64Byte的空间，4G肯定是远远不够的。

当然我们可能想到使用位图，每个URL取整数哈希值，置于位图相应的位置上。4G大概有320亿个bit，看上去是可行的。但位图适合对海量的、**取值分布很均匀**的集合去重。位图的空间复杂度是随集合内最大元素增大而线性增大的。要设计冲突率很低的哈希函数，势必要增加哈希值的取值范围，假如哈希值最大取到了2<sup>64</sup>，位图大概需要23亿G的空间。4G的位图最大值是320亿左右，为50亿条URL设计冲突率很低、最大值为320亿的哈希函数比较困难。

题目的一个解决思路是将文件切割成可以放入4G空间的小文件，重点在于A与B两个文件切割后的小文件要一一对应。

分别切割A与B文件，根据`hash(URL) % k`值将URL划分到k个不同的文件中，如A1，A2，...，Ak和B1，B2，...，Bk，同时可以保存hash值避免重复运算。这样Bn小文件与A文件共同的URL肯定会分到对应的An小文件中。读取An到一个哈希表中，再遍历Bn，判断是否有重复的URL。

另一个解决思路就是使用Bloom Filter布隆过滤器了。

### Bloom Filter简介

布隆过滤器(Bloom-Filter)是1970年由Bloom提出的。它可以用于检索一个元素是否在一个集合中。

布隆过滤器其实是位图的一种扩展，不同的是要使用多个哈希函数。它包括一个很长的二进制向量(位图)和一系列随机映射函数。

首先建立一个m位的位图，然后对于每一个加入的元素，使用k个哈希函数求k个哈希值映射到位图的k个位置，然后将这k个位置的bit全设置为1。下图是k=3的布隆过滤器：

![](./images/bloom-filter.jpg)

检索时，我们只要检索这些k个位是不是都是1就可以了：如果这些位有任何一个0，则被检元素一定不在；如果都是1，则被检元素很可能在。

可以看出布隆过滤器在时间和空间上的效率比较高，但也有缺点：

- 存在误判。布隆过滤器可以100%确定一个元素不在集合之中，但不能100%确定一个元素在集合之中。当k个位都为1时，也有可能是其它的元素将这些bit置为1的。
- 删除困难。一个放入容器的元素映射到位图的k个位置上是1，删除的时候不能简单的直接全部置为0，可能会影响其他元素的判断。

### Bloom Filter实现

要实现一个布隆过滤器，我们需要预估要存储的数据量为n，期望的误判率为P，然后计算位图的大小m，哈希函数的个数k，并选择哈希函数。

求位图大小m公式：

![](./images/bloom-filter-m.png)

哈希函数数目k公式：

![](./images/bloom-filter-k.png)

Python中已经有实现布隆过滤器的包：[pybloom](https://github.com/jaybaird/python-bloomfilter "pybloom")

安装
```
pip install pybloom
```

简单的看一下实现：

```python
class BloomFilter(object):
    FILE_FMT = b'<dQQQQ'

    def __init__(self, capacity, error_rate=0.001):
        """Implements a space-efficient probabilistic data structure
        capacity
            this BloomFilter must be able to store at least *capacity* elements
            while maintaining no more than *error_rate* chance of false
            positives
        error_rate
            the error_rate of the filter returning false positives. This
            determines the filters capacity. Inserting more than capacity
            elements greatly increases the chance of false positives.
        >>> b = BloomFilter(capacity=100000, error_rate=0.001)
        >>> b.add("test")
        False
        >>> "test" in b
        True
        """
        if not (0 < error_rate < 1):
            raise ValueError("Error_Rate must be between 0 and 1.")
        if not capacity > 0:
            raise ValueError("Capacity must be > 0")
        # given M = num_bits, k = num_slices, P = error_rate, n = capacity
        #       k = log2(1/P)
        # solving for m = bits_per_slice
        # n ~= M * ((ln(2) ** 2) / abs(ln(P)))
        # n ~= (k * m) * ((ln(2) ** 2) / abs(ln(P)))
        # m ~= n * abs(ln(P)) / (k * (ln(2) ** 2))
        num_slices = int(math.ceil(math.log(1.0 / error_rate, 2)))
        bits_per_slice = int(math.ceil(
            (capacity * abs(math.log(error_rate))) /
            (num_slices * (math.log(2) ** 2))))
        self._setup(error_rate, num_slices, bits_per_slice, capacity, 0)
        self.bitarray = bitarray.bitarray(self.num_bits, endian='little')
        self.bitarray.setall(False)

    def _setup(self, error_rate, num_slices, bits_per_slice, capacity, count):
        self.error_rate = error_rate
        self.num_slices = num_slices
        self.bits_per_slice = bits_per_slice
        self.capacity = capacity
        self.num_bits = num_slices * bits_per_slice
        self.count = count
        self.make_hashes = make_hashfuncs(self.num_slices, self.bits_per_slice)

    def __contains__(self, key):
        """Tests a key's membership in this bloom filter.
        >>> b = BloomFilter(capacity=100)
        >>> b.add("hello")
        False
        >>> "hello" in b
        True
        """
        bits_per_slice = self.bits_per_slice
        bitarray = self.bitarray
        hashes = self.make_hashes(key)
        offset = 0
        for k in hashes:
            if not bitarray[offset + k]:
                return False
            offset += bits_per_slice
        return True
```

计算公式基本一致。

算法将位图分成了k段(代码中的`num_slices`，也就是哈希函数的数量k)，每段长度为代码中的`bits_per_slice`，每个哈希函数只负责将对应的段中的bit置为1：

```python
        for k in hashes:
            if not skip_check and found_all_bits and not bitarray[offset + k]:
                found_all_bits = False
            self.bitarray[offset + k] = True
            offset += bits_per_slice
```

当期望误判率为0.001时，m与n的比率大概是14：

```python
>>> import math
>>> abs(math.log(0.001))/(math.log(2)**2)
14.37758756605116
```

当期望误判率为0.05时，m与n的比率大概是6：

```python
>>> import math
>>> abs(math.log(0.05))/(math.log(2)**2)
6.235224229572683
```

上述题目中，m最大为320亿，n为50亿，误判率大概为0.04，在可以接受的范围：
```python
>>> math.e**-((320/50.0)*(math.log(2)**2))
0.04619428041606246
```

### 应用

布隆过滤器一般用于在大数据量的集合中判定某元素是否存在：

1.**缓存穿透**：

缓存穿透，是指查询一个数据库中不一定存在的数据。正常情况下，查询先进行缓存查询，如果key不存在或者key已经过期，再对数据库进行查询，并将查询到的对象放进缓存。如果每次都查询一个数据库中不存在的key，由于缓存中没有数据，每次都会去查询数据库，很可能会对数据库造成影响。

缓存穿透的一种解决办法是为不存在的key缓存一个空值，直接在缓存层返回。这样做的弊端就是缓存太多空值占用了太多额外的空间，这点可以通过给缓存层空值设立一个较短的过期时间来解决。

另一种解决办法就是使用布隆过滤器，查询一个key时，先使用布隆过滤器进行过滤，如果判断请求查询key值存在，则继续查询数据库；如果判断请求查询不存在，直接丢弃。

2.**爬虫**：

在网络爬虫中，用于URL去重策略。

3.**垃圾邮件地址过滤**

由于垃圾邮件发送者可以不停地注册新地址，垃圾邮件的Email地址是一个巨量的集合。使用哈希表存贮几十亿个邮件地址可能需要上百GB的内存，而布隆过滤器只需要哈希表1/8到1/4的大小就能解决问题。布隆过滤器决不会漏掉任何一个在黑名单中的可疑地址。至于误判问题，常见的补救办法是在建立一个小的白名单，存储那些可能被误判的清白邮件地址。

4.Google的**BigTable**

Google的BigTable也使用了布隆过滤器，以减少不存在的行或列在磁盘上的I/O。

5.**Summary Cache**

[Summary Cache](http://pages.cs.wisc.edu/~jussara/papers/00ton.pdf)是一种用于代理服务器Proxy之间共享Cache的协议。可以使用布隆过滤器构建Summary Cache，每一个Cache的网页由URL唯一标识，因此Proxy的Cache内容可以表示为一个URL列表。进而我们可以将URL列表这个集合用布隆过滤器表示。

### 扩展

要实现删除元素，可以采用Counting Bloom Filter。它将标准布隆过滤器位图的每一位扩展为一个小的计数器(Counter)，插入元素时将对应的k个Counter的值分别加1，删除元素时则分别减1:

![](./images/bloom-filter-count.jpg)

代价就是多了几倍的存储空间。