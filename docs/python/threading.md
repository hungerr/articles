# Python并发编程

## 怎样修改全局变量是线程安全的？
Python VM 内部会使用 global interpreter lock （GIL）来确保同一时间只有一个线程运行。通常 Python 只会在字节码指令之间切换线程；切换的频率可以通过设置 sys.setswitchinterval() 指定。从 Python 程序的角度来看，每一条字节码指令以及每一条指令对应的 C 代码实现都是原子的。

理论上说，具体的结果要看具体的 PVM 字节码实现对指令的解释。而实际上，对内建类型（int，list，dict 等）的共享变量的“类原子”操作都是原子的。

举例来说，下面的操作是原子的（L、L1、L2 是列表，D、D1、D2 是字典，x、y 是对象，i，j 是 int 变量）：
```PYTHON
L.append(x)
L1.extend(L2)
x = L[i]
x = L.pop()
L1[i:j] = L2
L.sort()
x = y
x.field = y
D[x] = y
D1.update(D2)
D.keys()
```
这些不是原子的：
```PYTHON
i = i+1
L.append(L[-1])
L[i] = L[j]
D[x] = D[x] + 1
```
覆盖其他对象的操作会在其他对象的引用计数变成 0 时触发其 __del__() 方法，这可能会产生一些影响。对字典和列表进行大量操作时尤其如此。如果有疑问的话，使用互斥锁！