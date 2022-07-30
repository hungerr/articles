## lsof
`lsof(list openfiles)`，列出活跃进程的所有打开文件。

在Unix中一切（包括网络socket）都是文件。

### 输出

```bash
COMMAND PID TID    USER   FD   TYPE DEVICE   SIZE              NODE NAME
init      1        root  cwd    DIR    0,2    512  2814749767576442 /
init      1        root  rtd    DIR    0,2    512  2814749767576442 /
init      1        root  txt    REG    0,2 632096  3096224744287104 /init
```
lsof输出每列含义如下:

    COMMAND：进程的名称
    PID：进程标识符
    TID：任务 ID。Linux 下 TID 为空表示该行为进程
    USER：进程所有者
    FD：文件描述符
        初始打开每个应用程序时，都具有三个文件描述符，从 0 到 2，分别表示标准输入、输出和错误流。所以大多数应用程序所打开的文件的 FD 都是从 3 开始。
        主要有：
        cwd：应用程序当前工作目录，这是该应用程序启动的目录，除非它本身对这个目录进行更改
        txt：该类型的文件是程序代码，如应用程序二进制文件本身或共享库，如上列表中显示的 /sbin/init 程序
        lnn：库引用（AIX）
        err：FD 信息错误
        jld：监狱目录（FreeBSD）
        ltx：共享库文本（代码和数据）
        mxx：十六进制内存映射类型号 xx
        m86：DOS合并映射文件
        mem：内存映射文件
        mmap：内存映射设备
        pd：父目录
        rtd：根目录
        tr：内核跟踪文件（OpenBSD）
        v86：VP/ix 映射文件
        0：标准输出
        1：标准输入
        2：标准错误
        文件描述符后一般还跟着文件状态模式：
        r：只读模式
        w：写入模式
        u：读写模式
        空格：文件的状态模式为 unknow，且没有锁定
        -：文件的状态模式为 unknow，且被锁定
        
        同时在文件状态模式后面，还跟着相关的锁：
        N：对于未知类型的 Solaris NFS 锁
        r：文件部分的读锁
        R：整个文件的读锁
        w：文件的部分写锁
        W：整个文件的写锁
        u：任何长度的读写锁
        U：用于未知类型的锁
        x：用于部分文件上的 SCO OpenServer Xenix 锁
        X：用于整个文件上的 SCO OpenServer Xenix 锁
        space：无锁
    TYPE：文件类型。常见的文件类型有：
        REG：普通文件
        DIR：表示目录
        CHR：表示字符类型
        BLK：块设备类型
        UNIX：UNIX 域套接字
        FIFO：先进先出队列
        IPv4：IPv4 套接字
    DEVICE：磁盘名称
    SIZE：文件的大小或文件偏移量（以字节为单位）
    NODE：索引节点
    NAME：打开文件的确切名称

### 选项

- 默认 : 没有选项，lsof列出活跃进程的所有打开文件
- 组合 : 可以将选项组合到一起，如-abc，但要当心哪些选项需要参数
- `-a` : 结果进行**与**运算（而不是**或**）
- `-l` : 在输出显示用户ID而不是用户名
- `-h` : 获得帮助
- `-t` : 仅获取进程ID
- `-U` : 获取UNIX套接口地址
- `-F` : 格式化输出结果，用于其它命令。可以通过多种方式格式化，如-F pcfn（用于进程id、命令名、文件描述符、文件名，并以空终止

### 使用-i显示所有网络连接

语法: `lsof -i[46] [protocol][@hostname|hostaddr][:service|port]`

service is an /etc/services name - e.g., `smtp` - or a list of them.

#### 使用-i 6仅获取IPv6流量
```bash
lsof -i6
```

#### 仅显示TCP连接（同理可获得UDP连接）
```bash
lsof -iTCP
```

#### 使用-i:port来显示与指定端口相关的网络信息
```bash
lsof -i :22
```

#### 使用@host来显示指定到指定主机的连接
```bash
lsof -i@172.16.12.5
```

#### 使用@host:port显示基于主机与端口的连接
```bash
lsof -i @172.16.12.5:22
```

#### service或者port列表
```BASH
lsof -i tcp@foo:1-10,smtp,99
```

#### 找出监听端口
```bash
lsof -i -sTCP:LISTEN
```

#### 找出已建立的连接
```bash
lsof -i -sTCP:ESTABLISHED
```
你也可以通过grep搜索“ESTABLISHED”来完成该任务。
```BASH
lsof -i | grep -i ESTABLISHED
```

### 用户信息
你也可以获取各种用户的信息，以及它们在系统上正干着的事情，包括它们的网络活动、对文件的操作等。

#### 使用-u显示指定用户打开了什么
```BASH
lsof -u USERNAME/UID
```

#### 使用-u ^user来显示除指定用户以外的其它所有用户所做的事情
```BASH
lsof -u ^daniel
```

#### 杀死指定用户所做的一切事情
```BASH
kill -9 `lsof -t -u daniel`
```

### 命令和进程
可以查看指定程序或进程由什么启动

#### 使用-c查看指定的命令正在使用的文件和网络连接
```BASH
lsof -c init

COMMAND PID USER   FD   TYPE DEVICE   SIZE             NODE NAME
init      1 root  cwd    DIR    0,2    512 2814749767576442 /
init      1 root  rtd    DIR    0,2    512 2814749767576442 /
init      1 root  txt    REG    0,2 632096 3096224744287104 /init
```

#### 使用-p查看指定进程ID已打开的内容
```BASH
lsof -p 1
```

#### -t选项只返回PID
```BASH
lsof -t -c init

1
11
```

### 文件和目录
通过查看指定文件或目录，你可以看到系统上所有正与其交互的资源——包括用户、进程等。

#### 显示与指定目录交互的所有一切
```BASH
lsof /var/log/messages/
```

#### 显示与指定文件交互的所有一切
```BASH
lsof /home/daniel/firewall_whitelist.txt
```

#### +d显示目录下被进程开启的文件
```BASH
lsof +d /usr/local/ 
```

#### +D搜索目录下的目录
```BASH
lsof +D /usr/local/
```

#### -d显示使用fd进程
```
lsof -d 4
```

#### lsof +|-r [t] 控制lsof不断重复执行，缺省是15s刷新

`-r`，lsof会永远不断的执行，直到收到中断信号

`+r`，lsof会一直执行，直到没有档案被显示

不断查看目前ftp连接的情况：
```BASH
lsof -i tcp@ohaha.ks.edu.tw:ftp -r
```

#### lsof -s 列出打开文件的大小，如果没有大小，则留下空白

### 高级用法
与tcpdump类似，当你开始组合查询时，它就显示了它强大的功能。

#### 显示daniel连接到1.1.1.1所做的一切
```BASH
lsof -u daniel -i @1.1.1.1
```

#### 同时使用-t和-c选项以给进程发送 HUP 信号
```BASH
kill -HUP `lsof -t -c sshd`
```
#### lsof +L1显示所有打开的链接数小于1的文件

这通常（当不总是）表示某个攻击者正尝试通过删除文件入口来隐藏文件内容。

```BASH
lsof +L1
```

#### 显示某个端口范围的打开的连接
```BASH
lsof -i @fw.google.com:2150-2180
```

### `Lsof`诊断**Device is busy**问题
在系统运维过程中，我们会遇到磁盘卸载报**umount: /home: device is busy**之类的提示，如下我们给`/home`单独挂载了磁盘`/dev/sdb1`

卸载磁盘的时候提示如下
```BASH
[root@localhost ~]# umount /home/
umount: /home: device is busy.
        (In some cases useful info about processes that use
         the device is found by lsof(8) or fuser(1))
```
我们可以使用`lsof`命令找出占用磁盘的进程
```BASH
[root@localhost ~]# lsof /home
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
bash    5689 root  cwd    DIR   8,17     4096 131073 /home/123
```
直接kill掉此进程，磁盘成功卸载
```BASH
[root@localhost ~]# kill -9 5689
[root@localhost ~]# umount /home/
[root@localhost ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/sda2              18G  2.3G   15G  14% /
tmpfs                 491M  432K  491M   1% /dev/shm
/dev/sda1             291M   33M  244M  12% /boot
```
当然除了**kill**进程外也可以使用如下方法：
```BASH
umount -l /home 强行解除挂载

fuser -mv -k /homeh直接杀死占用磁盘的进程
```

### `Lsof`解决文件已删除空间未释放问题

磁盘空间达到100%会导致数据无法正常写入文件造成程序异常，如下/home空间达到100%，一般的解决方法是先删除大文件，然后有必要的情况下对磁盘进行扩容。

按照排查步骤，先进入/home目录查找占用磁盘的大文件（命令`du -h --max-depth=1 /home`），如下发现/home目录下并没有占用空间大的文件和目录。

那么为什么磁盘空间显示占用100%呢？原因是磁盘上文件已经被删除，但是存在进程仍然占用这些文件，因此其原来占用的磁盘空间并没有被释放，当然重启操作系统，空间会被释放出来，但是重启并不是最好的解决方法。

`lsof -n | grep delete`检查出占用文件的进程
```BASH
[root@192 home]# lsof -n | grep delete
gnome-ter 2770     zhang   22u      REG                8,2     16288     912130 /tmp/vte72PZM0 (deleted)
gnome-ter 2770     zhang   23u      REG                8,2      6816     913231 /tmp/vteMVOZM0 (deleted)
gnome-ter 2770     zhang   26u      REG                8,2       256     913233 /tmp/vteI5J3O0 (deleted)
tail      8400      root    3r      REG               8,17 714913792     131075 /home/123/Visio_Standard_2016_64Bit_ChnSimp.ISO (deleted)
tail      8406      root    3r      REG               8,17 714913792     131077 /home/123/345/Visio_Standard_2018_64Bit_ChnSimp.ISO (deleted)
tail      8406      root    4r      REG               8,17 714913792     131078 /home/123/345/Visio_Standard_2019_64Bit_ChnSimp.ISO (deleted)
tail      8406      root    5r      REG               8,17 714913792     131079 /home/123/345/Visio_Standard_2020_64Bit_ChnSimp.ISO (deleted)
tail      8415      root    3r      REG               8,17 714913792     131083 /home/123/zhang/Visio_Standard_19_64Bit73489nSimp.ISO (deleted)
tail      8415      root    4r      REG               8,17 714913792     131081 /home/123/zhang/Visio_Standard_2020_64Bit73489nSimp.ISO (deleted)
tail      8415      root    5r      REG               8,17 714913792     131082 /home/123/zhang/Visio_Standard_20_64Bit73489nSimp.ISO (deleted)

[root@192 home]# ll /home/123/
total 8
drwxr-xr-x. 2 root root 4096 Aug 15 23:05 345
lrwxrwxrwx. 1 root root    8 Jul  1 02:40 rh -> /opt/rh/
drwxr-xr-x. 2 root root 4096 Aug 15 23:05 zhang
[root@192 home]# ll /home/123/345/
total 0
```
从进程显示上可以看出，/home目录下占用文件已经删除，仍然有3个进程在占用，我们采用kill方式杀死进程，释放空间。
```BASH
[root@192 home]# kill -9 8400 8406 8415
[root@192 home]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/sda2              18G  2.4G   15G  14% /
tmpfs                 491M  432K  491M   1% /dev/shm
/dev/sda1             291M   33M  244M  12% /boot
/dev/sdb1             5.0G  139M  4.6G   3% /home
```

### 恢复删除的文件
当进程打开了某个文件时，只要该进程保持打开该文件，即使将其删除，它依然存在于磁盘中。这意味着，进程并不知道文件已经被删除，它仍然可以向打开该文件时提供给它的文件描述符进行读取和写入

在`/proc`目录下，其中包含了反映内核和进程树的各种文件。`/proc`目录挂载的是在内存中所映射的一块区域，所以这些文件和目录并不存在于磁盘中，因此当我们对这些文件进行读取和写入时，实际上是在从内存中获取相关信息。大多数与`lsof`相关的信息都存储于以进程的`PID`命名的目录中，即`/proc/1234`中包含的是`PID`为`1234`的进程的信息。
```BASH
lsof |grep /var/log/messages

syslogd 1283 root 2w REG 3,3 5381017 1773647 /var/log/messages (deleted)
```
从上面的信息可以看到`PID 1283（syslogd）`打开文件的文件描述符为`2`。同时还可以看到`/var/log/messages`已经标记被删除了。因此我们可以在`/proc/1283/fd/2` 中查看相应的信息，如下：
```BASH
head -n 10 /proc/1283/fd/2
```
从上面的信息可以查看要恢复的数据。如果可以通过文件描述符查看相应的数据，那么就可以使用 I/O 重定向将其复制到文件中，如:
```BASH
cat /proc/1283/fd/2 > /var/log/messages
```
