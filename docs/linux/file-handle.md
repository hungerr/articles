## 文件句柄

句柄的英文是 handle。在英文中，有操作、处理、控制之类的意义。作为一个名词时，是指某个中间媒介，通过这个中间媒介可控制、操作某样东西。

这样说有点抽象，举个例子。door handle 是指门把手，通过门把手可以去控制门，但 door handle 并非 door 本身，只是一个中间媒介。又比如 knife handle 是刀柄，通过刀柄可以使用刀。跟 door handle 类似，我们可以用 file handle 去操作 file, 但 file handle 并非 file 本身。这个 file handle 就被翻译成文件句柄，同理还有各种资源句柄。

计算机领域很多英文词，直接从日常词中引申而来。比如 fork，日常用词就是个叉子，在 unix 中引申成创建新进程（进程分叉了）。socket 日常用词是插座（连起来用于通电），引申成联网的标记信息（连起来用于通信）。英文是很日常，很容易理解的词，有时翻译成中文反而难以理解了。

具体到代码实现，handle 通常是某个数字标记，通过标记操作资源。这个标记在不同的场合有不同的叫法，有时叫 ID，有时叫描述符(descriptor)。在 Windows 平台，就叫各种 handle 了。

不要将 handle 简单地理解成编号、索引。比如分配 16 位的索引，再用 8 位密码将 16 位索引加密。之后将 4 位类型、4 位权限、8 位密码、16 位加密索引打包成一个 32 位的整数作为 handle。这时说这个 handle 是索引就有点不适当了。

用 handle 如何操作真正的资源，是实现的细节。handle 通常被实现为整数，也可以被实现成其他类型。

广义来说，指针也是某种 handle，可以操作对象。但实际语境中，指针跟句柄是有区别的。初次接触到 handle (或者 id)，很多人会有迷惑，为什么要用 handle，而不直接用指针呢？

- 指针作用太强，可做的事情太多。可做的事情越多，就会越危险。接口设计中，功能刚刚好就够了，并非越多权限越好的。
- handle 通常只是个整数，实现被隐藏起来，假如直接暴露了指针，也就暴露了指针类型（有时也可以暴露 void* 指针作为某种 handle）。用户看到越多细节，其代码就越有可能依赖这些细节。将来情况有变，但又要兼容用户代码，库内部改起来就更麻烦。
- 资源在内部管理，通过 handle 作为中间层，可以有效判断 handle 是否合法，也可以通过权限检查防止某种危险操作。
- handle 通常只是个整数，所有的语言都有整数这种类型，但并非所有语言都有指针。接口只出现整数，方便同一实现绑定到各种语言。

在普通的文件系统中，我们用文件索引节点编号(ino)表示一个文件。ino就是一个数字，ino保存在磁盘中，整个文件系统中任何两个文件的ino都不相同，因此给定一个ino，我们就能找到对应的文件。当使用NFS文件系统时就出现问题了，我们无法通过文件索引编号找到对应的文件。下面的例子中我们将一个文件系统挂载在另一个文件系统之上导出了。

因此，当NFS客户端给出一个文件索引节点编号时，服务器端无法确定到底是哪个文件系统中的索引编号，也就无法找到对应的文件。为了区分不同的文件系统，NFS用文件句柄标识一个文件，文件句柄中既包含了服务器端文件系统的信息，也包含了文件的信息。服务器端解析客户端传递过来的文件句柄，定位客户端请求的文件。对NFS客户端来说，文件句柄是透明的，客户端不关心文件句柄的构成方式，也不对文件句柄进行解析。只需要将文件句柄传递给服务器端就可以了。服务器端可以向文件句柄中加入任何信息，只要保证能根据文件句柄查找到对应的文件就可以了。

### 文件句柄的数据结构

```C
struct knfsd_fh {  
        // 文件句柄的长度  
        unsigned int    fh_size;        /* significant for NFSv3. 
                                         * Points to the current size while building 
                                         * a new file handle 
                                         */  
        union {  
                struct nfs_fhbase_old   fh_old;  
                __u32                   fh_pad[NFS4_FHSIZE/4];  
                struct nfs_fhbase_new   fh_new;  
        } fh_base;      // 文件句柄内容  
}; 
```
nfs_fhbase_new结构如下:
```
struct nfs_fhbase_new {  
        __u8            fb_version;             // 1      /* == 1, even => nfs_fhbase_old */  
        __u8            fb_auth_type;  
        __u8            fb_fsid_type;             
        __u8            fb_fileid_type;           
        __u32           fb_auth[1];               
/*      __u32           fb_fsid[0]; floating */  
/*      __u32           fb_fileid[0]; floating */  
};
```
- fb_version表示版本号，固定为1。

- fb_auth_type表示文件句柄是否经过了MD5校验，0表示没有经过校验，1表示进行了校验处理。目前的实现方式中固定为0。
- fb_fsid_type表示fsid的构成方式，fsid表示一个文件系统。
- fb_fileid_type表示fileid的构成方式，fileid表示一个文件。
- fb_auth 如果fb_auth_type为1，fb_auth表示文件句柄MD5校验值。如果fb_auth_type为0，则没有这个字段。
- fb_fsid 是一个文件系统的标识，这个字段的长度可变，根据fb_fsid_type进行设置。
- fb_fileid 是一个文件的标识，这个字段的长度也可变，根据fb_fileid_type进行设置。
```

fsid标识了文件系统中一个具体的文件:
struct fid {  
        union {  
                struct {  
                        u32 ino;  
                        u32 gen;  
                        u32 parent_ino;  
                        u32 parent_gen;  
                } i32;  
                struct {  
                        u32 block;  
                        u16 partref;  
                        u16 parent_partref;  
                        u32 generation;  
                        u32 parent_block;  
                        u32 parent_generation;  
                } udf;  
                __u32 raw[0];  
        };  
}; 
```

EXT2文件系统超级块结构ext2_sb_info中存在一个变量`s_next_generation`，每创建一个新文件，这个变量的值为赋给文件索引节点结构inode中的`i_generation`，同时s_next_generation加一。i_generation相等于文件的一个验证信息。如果用户删除了EXT2文件系统中一个文件，然后再创建一个同名文件，新创建文件的ino编号很可能和原文件ino编号相同，但是i_generation已经发生了变化。由于文件句柄中包含了i_generation，因此NFS文件系统可以检查出文件是否还是原来的文件，如果不是原来的文件，则NFS返回错误码`NFS3ERR_STALE(NFSv3)`，表示文件句柄已经过期了。


### NFS

根据Unix的规则，只要文件还在open状态（有object指向它），那么执行rm操作只会删除文件路径并将文件标记为"deleted"，文件的内容并不会从磁盘移除，已经打开文件的进程依然可以继续访问文件（参考这篇文章）。

但NFS有所不同，假设Client B对共享文件进行了rename/move操作，Client A依然可以访问这个文件，因为访问基于的是inode而不是文件名，但如果Client C在server端把文件rm了，之后A的读取操作就会失败。

究其原因，同样因为stateless。作为server，它并不知道文件被remove的时候，有没有client端的进程还在"open"这个文件，client端的close()操作在server端也没有对应的实现。

那Client A的"read error"又是怎么来的（怎么被server判断的）呢？文件被Client C移除后，对应的inode也将随之被server回收，并且可能很快就被reuse以创建新的文件，但Client A不知道啊，它还继续发送read这个inode的请求。

这时的inode已经不是Client A想要的那个inode了，看来啊，得对inode加上一些额外的信息，比如一个单调递增的数字，或者是文件创建时的timestamp，反正能作为唯一性的区别就好。NFS选择的是递增数字的方式，称为`generation number`，包在前面讲的`file handle`里，在之后client和server的交互中传递，过期的handle意味着文件已不复存在。

