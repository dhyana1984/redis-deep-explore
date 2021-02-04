
# keys 的问题
由于众所周知的原因，keys 命令不能随便使用，尤其是在 redis 的实例有成千上万个的时候，使用 keys 会全量遍历导致阻塞其他线程造成线上业务卡顿。

<!-- more -->

```sh
127.0.0.1:6379> set codehole1 a
OK
127.0.0.1:6379> set codehole2 b
OK
127.0.0.1:6379> set codehole3 c
OK
127.0.0.1:6379> set code1hole a
OK
127.0.0.1:6379> set code2hole b
OK
127.0.0.1:6379> set code3hole c
OK
127.0.0.1:6379> keys *
1) "code2hole"
2) "codehole2"
3) "code1hole"
4) "codehole1"
5) "code3hole"
6) "company"
7) "codehole3"
127.0.0.1:6379> keys codehole*
1) "codehole2"
2) "codehole1"
3) "codehole3"
127.0.0.1:6379> keys code*hole
1) "code2hole"
2) "code1hole"
3) "code3hole"
127.0.0.1:6379>
```

【keys 的缺陷】

1. 没有 offset、limit 参数，一次性吐出所有满足条件的 key，如果实例中有几百万个 key 满足条件，将会是满屏的字符串刷屏；
2. keys 算法是遍历算法，复杂度是 O(n)，如果实例中有千万级的 key，这个指令会导致 redis 服务卡顿，所有读写 redis 的其他指令都会被延后甚至超时报错，因为 redis 是单线程，顺序执行所有指令，其他指令必须等到当前的 keys 指令执行完才能继续。

【scan 的特点】

为了解决 keys 存在的问题，redis2.8 引入了 scan 指令：

1. 复杂度也是 O(n)，但它是通过游标分步进行的，不会阻塞线程；
2. 提供 limit 参数，可以控制每次返回的最大条数，注意：limit 只是一个略估值，实际返回的结果可多可少；
3. 和 keys 一样，也支持模式匹配功能；
4. 返回的结果可能会有重复，需要客户端去重；
5. 遍历的过程中如果有数据的修改，改动后的数据不确保能遍历到；
6. 单次返回的结果是空不代表遍历结束，要看游标的返回值是否为 0。

# scan 基本用法
> **scan cursor** [MATCH pattern] [COUNT count]

- cursor：游标整数值，首次遍历从 0 开始，直到返回结果为 0 表示遍历结束；
- MATCH pattern：匹配模式
- COUNT count：遍历的 limit

从以下输出可以看出，虽然指定了 count 为 1000，但是每次返回都只有十个左右，是因为这里的 count 不是指定返回结果的数量，而是限定 **单次遍历的字典槽位数量**。

```sh
127.0.0.1:6379> scan 0 match key99* count 1000
1) "10904"
2) 1) "key9952"
   2) "key999"
   3) "key9923"
   4) "key9964"
   5) "key9941"
   6) "key9903"
   7) "key9924"
   8) "key9986"
   9) "key9926"
127.0.0.1:6379> scan 10904 match key99* count 1000
1) "10828"
2)  1) "key9944"
    2) "key9988"
    3) "key9948"
    4) "key9906"
    5) "key9946"
    6) "key9955"
    7) "key9951"
    8) "key9910"
    9) "key994"
   10) "key9954"
   11) "key9992"
   12) "key9967"
127.0.0.1:6379> scan 10828 match key99* count 1000
1) "14802"
2)  1) "key9947"
    2) "key9990"
    3) "key9932"
    4) "key9975"
    5) "key9962"
    6) "key9933"
    7) "key9928"
    8) "key9991"
    9) "key9953"
   10) "key9930"
   11) "key9949"
   12) "key9968"
127.0.0.1:6379>
```

# 字典的结构
redis 中的所有 key 都存储在一个很大的字典中，这个字典的结构和 HashMap 一样，是个 `一维数组+链表` 的结构，如图：

![此处借用作者大大的图😁](http://img.yuzh.xyz/20200220121826_NdeNP2_Screenshot.png)

scan 指令返回的游标就是一维数组的位置索引，这个位置称之为「槽slot」，如果不考虑字典的扩容缩容，直接按数组下表挨个遍历就行了。count 参数表示要遍历的槽位数，之所以返回的结果可多可少，是因为不是所有的槽位上都会挂载链表，链表的挂载数量可能会有多个。每一次遍历都会将 count 数量上的槽位上挂载的所有链表元素进行模式匹配后过滤后，一次性返回给客户端。

# 其他 scan 指令
scan 不单能遍历所有的 key，还能遍历指定类型的 key。zsan 遍历 zset，hscan 遍历 hash，sscan 遍历 set。

# 大 key 扫描
在 redis 的使用过程中，要尽量避免大 key 产生。某个 key 太大会给数据迁移带来很大的问题，大 key 的扩容时会申请更大的内存，造成卡顿。

如果 redis 的内存大起大落，极有可能是因为大 key 导致的，这个时候需要定位到具体是哪个 key，做进一步的处理。对于定位线上的大 key，redis 官方提供了指令：

> redis-cli -h localhost -p 6379 --bigkeys

```sh
root@1dc111191659:/data# redis-cli -h localhost -p 6379 --bigkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest string found so far 'key9439' with 4 bytes
[16.37%] Biggest zset   found so far 'company' with 5 members

-------- summary -------

Sampled 10007 keys in the keyspace!
Total key length in bytes is 68951 (avg len 6.89)

Biggest string found 'key9439' has 4 bytes
Biggest   zset found 'company' has 5 members

0 lists with 0 items (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
10006 strings with 38896 bytes (99.99% of keys, avg size 3.89)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
1 zsets with 5 members (00.01% of keys, avg size 5.00)
root@1dc111191659:/data#
```
