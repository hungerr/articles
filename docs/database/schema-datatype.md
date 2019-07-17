## 范式与数据类型优化

良好的逻辑设计与物理设计是高性能的基石，应该根据系统将要执行的查询语句来设计schema。

### 优化数据类型

几个简单原则： 

- 更小的通常更好。一般情况下，尽量存储数据的最小数据类型，但要确保没有低估需要存储的值得范围。
- 简单就好：简单数据类型的操作通常需要更少的CPU周期。整型比字符操作代价更低，使用内建日期类型来存储日期，使用整数存储IP地址。MySQL提供INET_ATON()和INET_NTOA()函数在这两种表示方法之间转换。

施瓦茨(Baron Schwartz); 扎伊采夫(Peter Zaitsev); 特卡琴科(Vadim Tkachenko). 高性能MySQL(第3版) (Kindle 位置 3272-3273). 电子工业出版社. Kindle 版本. 
- 尽量避免NULL：最好指定列为NOT NULL。NULL列使得索引、索引统计和值都比较复杂。但InnoDB 使用单独的位（bit）存储NULL值，所以对于稀疏数据(很多值为NULL，只有少数行的列有非NULL值)有很好的空间效率。

选择数据类型时，第一步先确定合适的大类型：数字、字符串、时间等。下一步选择具体类型。

#### 整数类型

有几种整数类型：TINYINT(8)，SMALLINT(16)，MEDIUMINT(24)，INT(32)，BIGINT(64)。整数可以有UNSIGNED属性，不允许负值，可以将上限提高一倍。MySQL整数计算一般使用64位的BIGINT整数。

#### 实数类型

**FLOAT**(32)和**DOUBLE**(64)类型支持使用标准的浮点运算进行近似计算。MySQL内部使用DOUBLE作为内部浮点计算的类型。

**DECIMAL**类型用于存储精确的小数，支持精确计算。可以指定小数点前后所允许的最大位数。DECIMAL（18,9）小数点两边将各存储9个数 字，一共 使用9个字节：小数点前的数字用4个字节，小数点后的数字用4个字节，小数点本身占1个字节。

可以使用BIGINT代替DECIMAL，比如将货币单位精确到分，可以将元乘以100变为整数。

#### 字符串类型

**VARCHAR**：存储可变长字符串，最常见。它仅使用必要的空间，因此更节省空间。需要额外的1或2个字节记录长度。由于是变长的，UPDATE时需要做额外的工作。

适合VARCHAR的情况：

- 长度不平均，字符串列的最大长度比平均长度大很多
- 列的更新很少，碎片不是问题
- 使用了像UTF-8之类的复杂字符集，每个字符都使用不同的字节数进行存储

**CHAR**：定长

适合的情况：

- 很短的字符串，存储空间上更有效率，不需要额外记录长度
- 所有值都接近一个长度，比如密码的MD5值等
- 经常变更的数据

CHAR字符串末尾的空格会被截断(没有长度标识，定长，末尾都是空格)

**BINNARY**与**VARBINNARY**：存储的是二进制字符串。存储的是字节码而不是字符。BINARY采用的是`\0`字节而不是空格。

MySQL比较BINARY字符串时，每次按一个字节，并且根据该字节的数值进行比较。 因此，二进制比较比字符比较简单很多，所以也就更快。

更长的列会消耗更多的内存，所以更短的更好，只分配真正需要的空间。

**BLOB**和**TEXT**：为存储很大的数据而设计的字符串数据类型，分别采用二进制和字符方式存储。

与其他类型不同，MySQL把每个BLOB和TEXT值当作一个独立的对象处理。存储引擎在存储时通常会做特殊处理。当BLOB和TEXT值太大时，InnoDB会使用专门的**外部**存储区域来进行存储，此时每个值在行内需要1～4个字节存储一个指针，然后在外部存储区域存储实际的值。BLOB和TEXT家族之间仅有的不同是BLOB类型存储的是二进制数据，没有排序规则或字符集，而TEXT类型有字符集和排序规则。MySQL对BLOB和TEXT列进行排序与其他类型是不同的：它只对每个列的最前max_sort_length字节而不是整个字符串做排序。如果只需要排序前面一小部分字符，则可以减小max_sort_length的配置，或者使用ORDERBYSUSTRING（column，length）。MySQL不能将BLOB和TEXT列全部长度的字符串进行索引，也不能使用这些索引消除。

#### 枚举ENUM

可以使用ENUM代替常用字符串：
```
CREATE TABLE enum_test(
    e ENUM ('fish', 'apple', 'dog') NOT NULL
);
```
ENUM数据实际存储整数，而不是字符串。排序时也是按照内存存储的整数进行排序的。

枚举最不好的地方是，字符串列表是固定的，添加或者删除字符串必须使用ALTER TABLE。

### 日期与时间类型

MySQL能存储的最小时间粒度为秒，但也可以用微秒级别的粒度进行临时运算。

**DATETIME**：8个字节，能保存大范围的值，1001年到9999年，精度为秒，默认显示`2008-01-16 22:37:08`

**TIMESTAMP**：保存了从1970年1月1日午夜(格林乔治标准时间)以来的秒数，和UNIX时间戳相同。只用4个字节，范围小得多，只能表示从1970年至2038年。了 FROM_ UNIXTIME()函数将时间戳转换为日期，UNIX_ TIMESTAMP()函数将日期转换为Unix时间戳。

TIMESTAMP的显示依赖于时区。TIMESTAMP和DATETIME的行为将很不一样。前者提供的值与时区有关系，后者则保留文本表示的日期和时间。

TIMESTAMP也有DATETIME没有的特殊属性。默认情况下，如果插入时没有指定第一个TIMESTAMP列的值，MySQL则设置这个列的值为当前时间。在插入一行记录时，MySQL默认也会更新第一个TIMESTAMP列的值（除非在UPDATE语句中明确指定了值）。你可以配置任何TIMESTAMP列的插入和更新行为。最后，TIMESTAMP列默认为NOT NULL，这也和其他的数据类型不一样

尽量使用TIMESTAMP，空间效率更高。

### 位数据

技术上来说位数据都是字符串类型

#### BIT
在一列中存储一个或多个true/false值。BIT(1)存储1个位，BIT(2)存储2个位。最大位64个。

检索BIT时，是包含二进制0或者1值的字符串。在数字上下文的场景中检索时，结果将是位字符串转换成的数字。

存储`b'00111001'`(二进制57)到BIT(8)的列，检索时是字符码为57的字符串，也就是字符串`'9'`，数字上下文中得到数字57。

#### SET
保存很多true/false值。是一系列打包的位的集合。

保存权限的访问控制列表，每个位或者SET元素代表一个值：
```
CREATE TABLE acl (
    perms SET('CAN_READ','CAN_WRITE','CAN_DELETE') NOT NULL
);
INSERT INTO acl(perms) VALUES('CAN_READ','CAN_WRITE');
```

#### 选择标识符

整数类型通常是标识列最好的选择，可以AUTO_INCREMENT。避免使用字符串，因为索引的原因，会导致页分裂，磁盘随机访问，聚簇索引碎片等。

存储UUID值，可以移除`-`符号，或者使用UNHEX函数转换UUID为16字节的数组。

### schema设计

不要设计太多的列。

不要设计太多的关联，MySQL限制每个关联操作最多只能61张表，单个查询最好在12个表以内。

确实需要未知值得时候不要害怕使用NULL。

范式的数据库中，每个事实数据会出现并且只出现一次。反范式化的数据库中，信息是冗余的，可能会存储在多个地方。

#### 范式

优点：

- 更新操作比较快
- 只有很少或者没有重复数据
- 表通常更小

缺点：通常需要关联。

#### 反范式

可以很好的避免关联

实际应用中经常需要混用，可能使用部分范式化的schema、缓存表。

#### 缓存表

缓存表表示储存那些可以比较简单地从schema其他表获取数据的表(但获取速度比较慢)

#### 汇总表

保存的是使用GROUP BY语句聚合数据的表。

#### 物化视图

#### 计数器表

创建一个单独的表来记录点击次数。

单行会有互斥锁的限制，要想获取更高的并发量，可以保存在多行中：
```
CREATE TABLE hit_counter ( 
    slot tinyint unsigned not null primary key, 
    cnt int unsigned not null
) ENGINE= InnoDB;
```
预先在表中增加100行数据。随机选择一个槽进行更新：
```
UPDATE hit_counter 
SET cnt = cnt + 1 
WHERE slot = RAND() * 100;
```
统计：
```
SELECT SUM(cnt) FROM hit_counter;
```
每隔一段时间开始一个新的计数器：
```
CREATE TABLE daily_hit_counter ( 
    day date not null, 
    slot tinyint unsigned not null,
    cnt int unsigned not null, 
    primary key(day,slot) 
) ENGINE= InnoDB;
```
添加：
```
INSERT INTO daily_hit_counter(day, slot, cnt) 
VALUES(CURRENT_DATE, RAND() * 100, 1)  
ON DUPLICATE KEY UPDATE cnt = cnt + 1;
```
可以定时合并结果到0号槽，避免表变得更大：
```
UPDATE daily_ hit_ counter as c 
    INNER JOIN (SELECT day, SUM(cnt) AS cnt, MIN(slot)          AS mslot 
    FROM daily_hit_counter
    GROUP BY day ) AS x USING(day)
SET c.cnt = IF(c.slot = x.mslot, x.cnt, 0), 
    c.slot = IF(c.slot = x.mslot, 0, c.slot);

DELETE FROM daily_hit_counter 
WHERE slot <> 0 AND cnt = 0;
```

创建额外的索引、增加列、创建缓存表或者汇总表，会增加查询的负担。虽然写变慢了，但显著的提高了读操作的性能。

### ALTER TABLE

MySQL执行大部分修改表结构的操作方法是用新的结构创建一个空表，从旧表中查出所所有数据插入新表，然后删除旧表。通常很花时间。

可以先在一台不提供服务的机器上执行ALTER TABLE，然后进行切换。

另一种技巧是影子拷贝，用要求的表结构创建一张和源表完全无关的新表，然后通过重命名和删表操作互换。

MODIFY COLUMN操作都将导致表重建：
```
ALTER TABLE sakila. film 
MODIFY COLUMN rental_duration TINYINT(3) NOT NULL DEFAULT 5; 
```
ALTER COLUMN会直接修改.frm文件而不涉及表数据：
```
ALTER TABLE sakila.film 
ALTER COLUMN rental_duration SET DEFAULT 5;
```

### 只修改.frm文件

- 创建一张具有相同结构的空表，并进行所需要的修改
- 执行FLUSH TABLES WITH READ LOCK
- 交换.frm文件
- 执行UNLOCK TABLES

```
CREATE TABLE sakila.film_new LIKE sakila.film; 

ALTER TABLE sakila.film_new 
MODIFY COLUMN rating ENUM('G','PG','PG- 13','R',' NC-17', 'PG-14') DEFAULT 'G'; 

FLUSH TABLES WITH READ LOCK;
```