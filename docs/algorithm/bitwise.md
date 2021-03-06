## 位运算以及在算法中的应用

位运算是直接对整数在内存中的二进制位进行操作，比一般的算术运算符速度要快。

### 按位与(&)

相同位置的两个位都为1，则为1；若有一个不为1，则为0：

```
  1100
& 1010
------
  1000
```

`&`运算通常用于二进制的取位操作，例如一个数 and 1的结果就是取二进制的最末位。这可以用来判断一个整数的奇偶，二进制的最末位为0表示该数为偶数，最末位为1表示该数为奇数。


### 按位或(|)

相同位只要一个为1即为1。

or运算通常用于二进制特定位上的无条件赋值，例如一个数or 1的结果就是把二进制最末位强行变成1。如果需要把二进制最末位变成0，对这个数or 1之后再减一就可以了，其实际意义就是把这个数强行变成最接近的偶数。

### 按位异或(^)

如果某位不同则该位为1, 否则该位为0.

xor运算的逆运算是它本身，也就是说两次异或同一个数最后结果不变，即（a xor b) xor b = a。xor运算可以用于简单的加密，比如我想对我MM说1314520，但怕别人知道，于是双方约定拿我的生日19880516作为密钥。1314520 xor 19880516 = 20665500，我就把20665500告诉MM。MM再次计算20665500 xor 19880516的值，得到1314520。

### 按位反转(~)

### 左移(<<)

### 右移(>>)