# Redis中的LRU算法

`Redis`作为缓存使用时，一些场景下要考虑内存的空间消耗问题。`Redis`会删除过期键以释放空间，过期键的删除策略有两种：

- 惰性删除：每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话，就删除该键；如果没有过期，就返回该键。
- 定期删除：每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。

另外，`Redis`也可以开启`LRU`功能来自动淘汰一些键值对。

我们知道`LRU`算法一般会用一个双向链表记录键值对的顺序，但`Redis`出于空间考虑问题，并未使用一个全局双向链表管理所有的数据，而是使用了一种近似算法，就是随机取出若干个key，然后按照访问时间排序后，淘汰掉最不经常使用的。

## LRU配置参数
`Redis`配置中和`LRU`有关的有三个：

- `maxmemory`: 配置`Redis`存储数据时指定限制的内存大小，比如`100m`。当缓存消耗的内存超过这个数值时, 将触发数据淘汰。该数据配置为0时，表示缓存的数据量没有限制, 即LRU功能不生效。64位的系统默认值为0，32位的系统默认内存限制为3GB
- `maxmemory_policy`: 触发数据淘汰后的淘汰策略
- `maxmemory_samples`: 随机采样的精度，也就是随即取出key的数目。该数值配置越大, 越接近于真实的LRU算法，但是数值越大，相应消耗也变高，对性能有一定影响，样本值默认为5。

## 淘汰策略
淘汰策略即`maxmemory_policy`的赋值有以下几种：

- `noeviction`:如果缓存数据超过了`maxmemory`限定值，并且客户端正在执行的命令(大部分的写入指令，但DEL和几个指令例外)会导致内存分配，则向客户端返回错误响应
- `allkeys-lru`: 对所有的键都采取`LRU`淘汰
- `volatile-lru`: 仅对设置了过期时间的键采取`LRU`淘汰
- `allkeys-random`: 随机回收所有的键
- `volatile-random`: 随机回收设置过期时间的键
- `volatile-ttl`: 仅淘汰设置了过期时间的键---淘汰生存时间`TTL(Time To Live)`更小的键

`volatile-lru`, `volatile-random`和`volatile-ttl`这三个淘汰策略使用的不是全量数据，有可能无法淘汰出足够的内存空间。在没有过期键或者没有设置超时属性的键的情况下，这三种策略和`noeviction`差不多。

一般的经验规则:

- 使用`allkeys-lru`策略：当预期请求符合一个幂次分布(二八法则等)，比如一部分的子集元素比其它其它元素被访问的更多时，可以选择这个策略。
- 使用`allkeys-random`：循环连续的访问所有的键时，或者预期请求分布平均（所有元素被访问的概率都差不多）
- 使用`volatile-ttl`：要采取这个策略，缓存对象的`TTL`值最好有差异

`volatile-lru` 和 `volatile-random`策略，当你想要使用单一的`Redis`实例来同时实现缓存淘汰和持久化一些经常使用的键集合时很有用。未设置过期时间的键进行持久化保存，设置了过期时间的键参与缓存淘汰。不过一般运行两个实例是解决这个问题的更好方法。

为键设置过期时间也是需要消耗内存的，所以使用`allkeys-lru`这种策略更加节省空间，因为这种策略下可以不为键设置过期时间。

## 近似LRU算法
所以，出于节省内存的考虑，`Redis`的`LRU`算法并非完整的实现。`Redis`并不会选择最久未被访问的键进行回收，相反它会尝试运行一个近似`LRU`的算法，通过对少量键进行取样，然后回收其中的最久未被访问的键。

`Redis`3.0之后改善了算法的性能，会提供一个候选回收键`pool`，使得更加近似真实`LRU`算法的行为。

通过调整每次回收时的采样数量`maxmemory-samples`，可以实现调整算法的精度。

真实`LRU`算法与近似`LRU`的算法可以通过下面的图像对比：
![](./images/redis_lru_comparison.png)

浅灰色带是已经被淘汰的对象，灰色带是没有被淘汰的对象，绿色带是新添加的对象。可以看出，`maxmemory-samples`值为5时`Redis 3.0`效果比`Redis 2.8`要好。使用10个采样大小的`Redis 3.0`的近似`LRU`算法已经非常接近理论的性能了。

数据访问模式非常接近幂次分布时，也就是大部分的访问集中于部分键时，`LRU`近似算法会处理得很好。

在模拟实验的过程中，我们发现如果使用幂次分布的访问模式，真实`LRU`算法和近似`LRU`算法几乎没有差别。

## LFU模式







`server.h`
[https://github.com/antirez/redis/blob/5.0/src/server.h#L605](https://github.com/antirez/redis/blob/5.0/src/server.h#L605)
```
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

`db.c`
[https://github.com/antirez/redis/blob/5.0/src/db.c#L55](https://github.com/antirez/redis/blob/5.0/src/db.c#L55)

```
robj *lookupKey(redisDb *db, robj *key, int flags) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);

        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        if (server.rdb_child_pid == -1 &&
            server.aof_child_pid == -1 &&
            !(flags & LOOKUP_NOTOUCH))
        {
            if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
                updateLFU(val);
            } else {
                val->lru = LRU_CLOCK();
            }
        }
        return val;
    } else {
        return NULL;
    }
}
```

`server.c`
[https://github.com/antirez/redis/blob/5.0/src/server.c#L2545](https://github.com/antirez/redis/blob/5.0/src/server.c#L2545)
```
int processCommand(client *c) {
    moduleCallCommandFilters(c);

    /* The QUIT command is handled separately. Normal command procs will
     * go through checking for replication and QUIT will cause trouble
     * when FORCE_REPLICATION is enabled and would be implemented in
     * a regular command proc. */
    if (!strcasecmp(c->argv[0]->ptr,"quit")) {
        addReply(c,shared.ok);
        c->flags |= CLIENT_CLOSE_AFTER_REPLY;
        return C_ERR;
    }

    /* Now lookup the command and check ASAP about trivial error conditions
     * such as wrong arity, bad command name and so forth. */
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    if (!c->cmd) {
        flagTransaction(c);
        sds args = sdsempty();
        int i;
        for (i=1; i < c->argc && sdslen(args) < 128; i++)
            args = sdscatprintf(args, "`%.*s`, ", 128-(int)sdslen(args), (char*)c->argv[i]->ptr);
        addReplyErrorFormat(c,"unknown command `%s`, with args beginning with: %s",
            (char*)c->argv[0]->ptr, args);
        sdsfree(args);
        return C_OK;
    } else if ((c->cmd->arity > 0 && c->cmd->arity != c->argc) ||
               (c->argc < -c->cmd->arity)) {
        flagTransaction(c);
        addReplyErrorFormat(c,"wrong number of arguments for '%s' command",
            c->cmd->name);
        return C_OK;
    }

    /* Check if the user is authenticated. This check is skipped in case
     * the default user is flagged as "nopass" and is active. */
    int auth_required = !(DefaultUser->flags & USER_FLAG_NOPASS) &&
                        !c->authenticated;
    if (auth_required || DefaultUser->flags & USER_FLAG_DISABLED) {
        /* AUTH and HELLO are valid even in non authenticated state. */
        if (c->cmd->proc != authCommand || c->cmd->proc == helloCommand) {
            flagTransaction(c);
            addReply(c,shared.noautherr);
            return C_OK;
        }
    }

    /* Check if the user can run this command according to the current
     * ACLs. */
    int acl_retval = ACLCheckCommandPerm(c);
    if (acl_retval != ACL_OK) {
        flagTransaction(c);
        if (acl_retval == ACL_DENIED_CMD)
            addReplyErrorFormat(c,
                "-NOPERM this user has no permissions to run "
                "the '%s' command or its subcommnad", c->cmd->name);
        else
            addReplyErrorFormat(c,
                "-NOPERM this user has no permissions to access "
                "one of the keys used as arguments");
        return C_OK;
    }

    /* If cluster is enabled perform the cluster redirection here.
     * However we don't perform the redirection if:
     * 1) The sender of this command is our master.
     * 2) The command has no key arguments. */
    if (server.cluster_enabled &&
        !(c->flags & CLIENT_MASTER) &&
        !(c->flags & CLIENT_LUA &&
          server.lua_caller->flags & CLIENT_MASTER) &&
        !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0 &&
          c->cmd->proc != execCommand))
    {
        int hashslot;
        int error_code;
        clusterNode *n = getNodeByQuery(c,c->cmd,c->argv,c->argc,
                                        &hashslot,&error_code);
        if (n == NULL || n != server.cluster->myself) {
            if (c->cmd->proc == execCommand) {
                discardTransaction(c);
            } else {
                flagTransaction(c);
            }
            clusterRedirectClient(c,n,hashslot,error_code);
            return C_OK;
        }
    }

    /* Handle the maxmemory directive.
     *
     * Note that we do not want to reclaim memory if we are here re-entering
     * the event loop since there is a busy Lua script running in timeout
     * condition, to avoid mixing the propagation of scripts with the
     * propagation of DELs due to eviction. */
    if (server.maxmemory && !server.lua_timedout) {
        int out_of_memory = freeMemoryIfNeededAndSafe() == C_ERR;
        /* freeMemoryIfNeeded may flush slave output buffers. This may result
         * into a slave, that may be the active client, to be freed. */
        if (server.current_client == NULL) return C_ERR;

        /* It was impossible to free enough memory, and the command the client
         * is trying to execute is denied during OOM conditions or the client
         * is in MULTI/EXEC context? Error. */
        if (out_of_memory &&
            (c->cmd->flags & CMD_DENYOOM ||
             (c->flags & CLIENT_MULTI && c->cmd->proc != execCommand))) {
            flagTransaction(c);
            addReply(c, shared.oomerr);
            return C_OK;
        }
    }

    /* Don't accept write commands if there are problems persisting on disk
     * and if this is a master instance. */
    int deny_write_type = writeCommandsDeniedByDiskError();
    if (deny_write_type != DISK_ERROR_TYPE_NONE &&
        server.masterhost == NULL &&
        (c->cmd->flags & CMD_WRITE ||
         c->cmd->proc == pingCommand))
    {
        flagTransaction(c);
        if (deny_write_type == DISK_ERROR_TYPE_RDB)
            addReply(c, shared.bgsaveerr);
        else
            addReplySds(c,
                sdscatprintf(sdsempty(),
                "-MISCONF Errors writing to the AOF file: %s\r\n",
                strerror(server.aof_last_write_errno)));
        return C_OK;
    }

    /* Don't accept write commands if there are not enough good slaves and
     * user configured the min-slaves-to-write option. */
    if (server.masterhost == NULL &&
        server.repl_min_slaves_to_write &&
        server.repl_min_slaves_max_lag &&
        c->cmd->flags & CMD_WRITE &&
        server.repl_good_slaves_count < server.repl_min_slaves_to_write)
    {
        flagTransaction(c);
        addReply(c, shared.noreplicaserr);
        return C_OK;
    }

    /* Don't accept write commands if this is a read only slave. But
     * accept write commands if this is our master. */
    if (server.masterhost && server.repl_slave_ro &&
        !(c->flags & CLIENT_MASTER) &&
        c->cmd->flags & CMD_WRITE)
    {
        addReply(c, shared.roslaveerr);
        return C_OK;
    }

    /* Only allow a subset of commands in the context of Pub/Sub if the
     * connection is in RESP2 mode. With RESP3 there are no limits. */
    if ((c->flags & CLIENT_PUBSUB && c->resp == 2) &&
        c->cmd->proc != pingCommand &&
        c->cmd->proc != subscribeCommand &&
        c->cmd->proc != unsubscribeCommand &&
        c->cmd->proc != psubscribeCommand &&
        c->cmd->proc != punsubscribeCommand) {
        addReplyError(c,"only (P)SUBSCRIBE / (P)UNSUBSCRIBE / PING / QUIT allowed in this context");
        return C_OK;
    }

    /* Only allow commands with flag "t", such as INFO, SLAVEOF and so on,
     * when slave-serve-stale-data is no and we are a slave with a broken
     * link with master. */
    if (server.masterhost && server.repl_state != REPL_STATE_CONNECTED &&
        server.repl_serve_stale_data == 0 &&
        !(c->cmd->flags & CMD_STALE))
    {
        flagTransaction(c);
        addReply(c, shared.masterdownerr);
        return C_OK;
    }

    /* Loading DB? Return an error if the command has not the
     * CMD_LOADING flag. */
    if (server.loading && !(c->cmd->flags & CMD_LOADING)) {
        addReply(c, shared.loadingerr);
        return C_OK;
    }

    /* Lua script too slow? Only allow a limited number of commands. */
    if (server.lua_timedout &&
          c->cmd->proc != authCommand &&
          c->cmd->proc != helloCommand &&
          c->cmd->proc != replconfCommand &&
        !(c->cmd->proc == shutdownCommand &&
          c->argc == 2 &&
          tolower(((char*)c->argv[1]->ptr)[0]) == 'n') &&
        !(c->cmd->proc == scriptCommand &&
          c->argc == 2 &&
          tolower(((char*)c->argv[1]->ptr)[0]) == 'k'))
    {
        flagTransaction(c);
        addReply(c, shared.slowscripterr);
        return C_OK;
    }

    /* Exec the command */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }
    return C_OK;
}
```

`evict.c`
[https://github.com/antirez/redis/blob/5.0/src/evict.c#L446](https://github.com/antirez/redis/blob/5.0/src/evict.c#L446)

dictGetRandomKey dict.c
[https://github.com/antirez/redis/blob/5.0/src/dict.c#L610](https://github.com/antirez/redis/blob/5.0/src/dict.c#L610)

`evict.c`  LRU_CLOCK  estimateObjectIdleTime
[https://github.com/antirez/redis/blob/5.0/src/evict.c#L78](https://github.com/antirez/redis/blob/5.0/src/evict.c#L78)

serverCron
[https://github.com/antirez/redis/blob/5.0/src/server.c#L1091](https://github.com/antirez/redis/blob/5.0/src/server.c#L1091)