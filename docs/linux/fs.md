## Linux文件系统

### Ext2 (Linux second extended file system, ext2fs)

较新的操作系统的文件数据除了文件实际内容外， 通常含有非常多的属性，例如Linux操作系统的文件权限(rwx)与文件属性(拥有者、群组、时间参数等)。 文件系统通常会将这两部份的数据分别存放在不同的区块，权限与属性放置到`inode`中，至于实际数据则放置到`data block`区块中。 另外，还有一个超级区块`superblock`会记录整个文件系统的整体信息，包括`inode`与`block`的总量、使用量、剩余量等。

每个`inode`与`block`都有编号，至于这三个数据的意义可以简略说明如下：

- `superblock`：记录此filesystem的整体信息，包括`inode/block`的总量、使用量、剩余量， 以及文件系统的格式与相关信息等；
- `inode`：记录文件的属性，一个文件占用一个`inode`，同时记录此文件的数据所在的`block`号码；
- `block`：实际记录文件的内容，若文件太大时，会占用多个block

这种数据存取的方法我们称为`索引式文件系统indexed allocation`


`inode`的内容在记录文件的权限与相关属性，至于`block`区块则是在记录文件的实际内容。 而且文件系统一开始就将 inode 与 block 规划好了，除非重新格式化(或者利用 resize2fs 等命令变更文件系统大小)，否则 inode 与 block 固定后就不再变动。但是如果仔细考虑一下，如果我的文件系统高达数百GB时， 那么将所有的 inode 与 block 通通放置在一起将是很不智的决定，因为 inode 与 block 的数量太庞大，不容易管理。

为此之故，因此 Ext2 文件系统在格式化的时候基本上是区分为多个区块群组 (block group) 的，每个区块群组都有独立的 inode/block/superblock 系统

每一个区块群组(block group)的六个主要内容说明如后：

### Superblock (超级区块)
Superblock 是记录整个 filesystem 相关信息的地方， 没有 Superblock ，就没有这个 filesystem 了。他记录的信息主要有：

- block 与 inode 的总量；
- 未使用与已使用的 inode / block 数量；
- block 与 inode 的大小 (block 为 1, 2, 4K，inode 为 128 bytes)；
- filesystem 的挂载时间、最近一次写入数据的时间、最近一次检验磁盘 (fsck) 的时间等文件系统的相关信息；
- 一个 valid bit 数值，若此文件系统已被挂载，则 valid bit 为 0 ，若未被挂载，则 valid bit 为 1 。

### data block
data block 是用来放置文件内容数据地方，在 Ext2 文件系统中所支持的 block 大小有 1K, 2K 及 4K 三种而已。在格式化时 block 的大小就固定了，且每个 block 都有编号，以方便 inode 的记录

- 每个 block 内最多只能够放置一个文件的数据；
- 承上，如果文件大于 block 的大小，则一个文件会占用多个 block 数量；
- 承上，若文件小于 block ，则该 block 的剩余容量就不能够再被使用了(磁盘空间会浪费)。

### inode
各个inode项具有唯一的编号 (该文件系统唯一)，即inode number

node 的内容在记录文件的属性以及该文件实际数据是放置在哪几号 block 内

inode 记录的文件数据至少有底下这些：(注4)

- 该文件的存取模式(read/write/excute)；
- 该文件的拥有者与群组(owner/group)；
- 该文件的大小；
- 文件的时间戳，共有三个：ctime指inode上一次变动的时间，mtime指文件内容上一次变动的时间，atime指文件上一次打开的时间。
- 定义文件特性的旗标(flag)，如 SetUID...；
- 该文件真正内容的指向 (pointer)；
- 链接数，即有多少文件名指向这个inode
- 文件数据block的位置

注意：inode中不存储文件名数据

inode 的数量与大小也是在格式化时就已经固定了，除此之外

- 每个 inode 大小均固定为 128 bytes；
- 每个文件都仅会占用一个 inode 而已；
- 承上，因此文件系统能够创建的文件数量与 inode 的数量有关；
- 系统读取文件时需要先找到 inode，并分析 inode 所记录的权限与用户是否符合，若符合才能够开始实际读取 block 的内容。

这里值得重复一遍，Unix/Linux系统内部不使用文件名，而使用inode号码来识别文件。对于系统来说，文件名只是inode号码便于识别的别称或者绰号。

表面上，用户通过文件名，打开文件。实际上，系统内部这个过程分成三步：首先，系统找到这个文件名对应的inode号码；其次，通过inode号码，获取inode信息；最后，根据inode信息，找到文件数据所在的block，读出数据。

可以用stat命令，查看某个文件的inode信息：
```BASH
stat example.txt
```

### Filesystem Description (文件系统描述说明)
这个区段可以描述每个 block group 的开始与结束的 block 号码，以及说明每个区段 (superblock, bitmap, inodemap, data block) 分别介于哪一个 block 号码之间。这部份也能够用 dumpe2fs 来观察的。


### block bitmap (区块对照表)
从 block bitmap 当中可以知道哪些 block 是空的，因此我们的系统就能够很快速的找到可使用的空间来处置文件

同样的，如果你删除某些文件时，那么那些文件原本占用的 block 号码就得要释放出来， 此时在 block bitmap 当中相对应到该 block 号码的标志就得要修改成为『未使用中』

### inode bitmap (inode 对照表)
这个其实与 block bitmap 是类似的功能，记录使用与未使用的 inode 号码

### 目录与inode

当我们在 Linux 下的 ext2 文件系统创建一个目录时， ext2 会分配一个 inode 与至少一块 block 给该目录。其中，inode 记录该目录的相关权限与属性，并可记录分配到的那块 block 号码； 而 block 则是记录在这个目录下的文件名与该文件名占用的 inode 号码数据。

可以使用 `ls -i` 这个选项来处理：

inode 本身并不记录文件名，文件名的记录是在目录的 block 当中。 因此当我们要读取某个文件时，就务必会经过目录的 inode 与 block ，然后才能够找到那个待读取文件的 inode 号码， 最终才会读到正确的文件的 block 内的数据。

### inode数据结构

ext4 inode
```C
/*
 * Structure of an inode on the disk
 */
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size_lo;	/* Size in bytes */
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Inode Change time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */
	__le16	i_gid;		/* Low 16 bits of Group Id */
	__le16	i_links_count;	/* Links count */
	__le32	i_blocks_lo;	/* Blocks count */
	__le32	i_flags;	/* File flags */
	union {
		struct {
			__le32  l_i_version;
		} linux1;
		struct {
			__u32  h_i_translator;
		} hurd1;
		struct {
			__u32  m_i_reserved1;
		} masix1;
	} osd1;				/* OS dependent 1 */
	__le32	i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
	__le32	i_generation;	/* File version (for NFS) */
	__le32	i_file_acl_lo;	/* File ACL */
	__le32	i_size_high;
	__le32	i_obso_faddr;	/* Obsoleted fragment address */
	union {
		struct {
			__le16	l_i_blocks_high; /* were l_i_reserved1 */
			__le16	l_i_file_acl_high;
			__le16	l_i_uid_high;	/* these 2 fields */
			__le16	l_i_gid_high;	/* were reserved2[0] */
			__le16	l_i_checksum_lo;/* crc32c(uuid+inum+inode) LE */
			__le16	l_i_reserved;
		} linux2;
		struct {
			__le16	h_i_reserved1;	/* Obsoleted fragment number/size which are removed in ext4 */
			__u16	h_i_mode_high;
			__u16	h_i_uid_high;
			__u16	h_i_gid_high;
			__u32	h_i_author;
		} hurd2;
		struct {
			__le16	h_i_reserved1;	/* Obsoleted fragment number/size which are removed in ext4 */
			__le16	m_i_file_acl_high;
			__u32	m_i_reserved2[2];
		} masix2;
	} osd2;				/* OS dependent 2 */
	__le16	i_extra_isize;
	__le16	i_checksum_hi;	/* crc32c(uuid+inum+inode) BE */
	__le32  i_ctime_extra;  /* extra Change time      (nsec << 2 | epoch) */
	__le32  i_mtime_extra;  /* extra Modification time(nsec << 2 | epoch) */
	__le32  i_atime_extra;  /* extra Access time      (nsec << 2 | epoch) */
	__le32  i_crtime;       /* File Creation time */
	__le32  i_crtime_extra; /* extra FileCreationtime (nsec << 2 | epoch) */
	__le32  i_version_hi;	/* high 32 bits for 64-bit version */
	__le32	i_projid;	/* Project ID */
};
```

### 链接

#### 硬链接

要了解硬链接是什么，关键是要了解文件的标识是其inode编号，而不是其名称。 硬链接是一个引用inode的名称。 这意味着如果file1有一个名为file2的硬链接，那么这两个文件都引用相同的inode。 因此，当您为文件创建硬链接时，您真正要做的就是为inode添加一个新名称。 为此，请使用不带选项的`ln`命令。

链接计数是文件硬链接的数字。 因此，创建硬链接后链接计数links会增加，还有在目录文件对应数据块上增加一条文件名和inode对应关系记录；

只有将硬链接和原文件都删除之后，文件才会真正删除，即links为0才真正删除

一旦删除操作导致links数为0系统将回收inode编号，重用文件所在的簇

硬链接有两个限制：

- 目录不能硬链接。 Linux不允许这样做以维护目录的非循环树结构。
- 无法跨文件系统创建硬链接。 这两个文件必须位于相同的文件系统上，因为不同的文件系统具有不同的独立inode表（不同文件系统上具有相同inode编号的两个文件是不同的）

#### 软链接

软链接是一个单独的新的文件，其内容`block`指向被链接的文件。 要创建符号链接，请使用`ln -s`

### ext4

#### 新的inode
Ext4支持更大的inode。之前的Ext3默认的inode大小128字节，Ext4为了在inode中容纳更多的扩展属性，默认inode大小为256字节。另外，Ext4还支持快速扩展属性和inode保留

#### 大文件系统
ext3 文件系统使用 32 位寻址，这限制它仅支持 2 TiB 文件大小和 16 TiB 文件系统系统大小（这是假设在块大小为 4 KiB 的情况下，一些 ext3 文件系统使用更小的块大小，因此对其进一步被限制）。

ext4 使用 48 位的内部寻址，理论上可以在文件系统上分配高达 16 TiB 大小的文件，其中文件系统大小最高可达 1000000 TiB(1 EiB)

#### 无限制的子目录
ext3 仅限于 32000 个子目录；ext4 允许无限数量的子目录。从 2.6.23 内核版本开始，ext4 使用 HTree 索引来减少大量子目录的性能损失。