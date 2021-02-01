# 什么是布隆过滤器
使用 HyperLogLog 数据结构进行去重统计非常高效，但是如果我们想知道某个 value 是否在 HyperLogLog 结构里面这时它就无能为力了。它只提供了 pf.add 和 pd.count 方法，并没有 pf.contains 方法。
<!-- more -->
> 使用背景：新闻推送去重。每次推送新闻都需要去除我们看过的。传统的做法，我们会记录用户的历史记录，每次推送新内容时都去历史记录中筛选，过滤掉已经存在的记录。但是当用户量和新闻量很大时，这种做法性能是否跟得上？而且如果历史记录存储在数据库，需要频繁的做 exist 查询，在并发量很大的情况下，数据库很难扛住压力。如果将历史记录缓存在 redis，则需要很大的空间，存储空间会随着时间的推移越来越大，不是长久之计。

布隆过滤器就是用来解决这种去重问题的，它在去重的同时能节省 90% 的空间，只是有稍微的不精确，有一定几率出现误判。

# Redis 中的布隆过滤器
布隆过滤器可以理解为一个不怎么精确的 set 结构，当使用 contains 方法判断某个数据是否存在时，它可能出现误判。但也不是特别不精确，只要参数设置的合理，他的精确度可以控制的足够精确。

**使用布隆过滤器时，当它说某个值不存在，就一定不存在。它说某个值存在，有可能是不存在的。**

在新闻推荐的使用场景中，布隆过滤器能够过滤掉用户已经看过的新闻，而那些没有看过的新闻，有一小部分也会被过滤掉（误判）。但绝大多数内容它都能准确识别，这样可以保证推荐给用户的新闻都是新的。

# 食用方法
布隆过滤器在 redis4.0 中以插件的形式提供，跳过安装插件细节，直接使用带有插件的 docker 版本：

```sh
docker pull redislabs/rebloom
docker run -d -p 6379:6379 redislabs/rebloom
```

【基本指令】

- bf.add key ...options...：添加单个
- bf.exists key ...options...：单个是否存在
- bf.madd key ...options...：添加多个
- bf.mexists key ...options...：多个是否存在

```sh
127.0.0.1:6379> bf.add codehole user1
(integer) 1
127.0.0.1:6379> bf.exists codehole user1
(integer) 1
127.0.0.1:6379> bf.madd codehole user2 user3 user4 user5
1) (integer) 1
2) (integer) 1
3) (integer) 1
4) (integer) 1
127.0.0.1:6379> bf.mexists codehole user2 user3 user4 user5 user6
1) (integer) 1
2) (integer) 1
3) (integer) 1
4) (integer) 1
5) (integer) 0
```

从结果看来，并没有出现误判。是因为 **布隆过滤器不会误判已存在的value，而会误判不存在的。** 也就是说，如果 bf.exist 一个不存在的值，它有可能认为存在。为了查看是否出现误判，使用代码添加更多的元素来验证一下。

> jedis 目前没有提供指令拓展机制，无法使用 Redis Module 提供的 bf.xx 指令。可以使用 JreBloom 包，其提供了布隆过滤器指令的支持。

```java
public class BloomTest {
    public static void main(String[] args) {
        Client client = new Client("localhost", 6379);
        client.delete("codehole");
        for (int i = 0; i < 100000; i++) {
            client.add("codehole", "user" + i);
            // boolean ret = client.exists("codehole", "user" + i); 查找已存在的元素，不会误判。
            boolean ret = client.exists("codehole", "user" + (i + 1)); // 查找不存在的元素，会出现误判。
            if (ret) {
                System.out.println("到第 " + i + " 个元素时出现误判");
                break;
            }
        }
        client.close();
    }
}
```

输出结果：

    到第 310 个元素时出现误判

# 参数设置
bf.reserve 用来创建一个布隆过滤器，有三个参数：

- key：键名
- error_rate：默认是 0.01
- initial_size：默认是 100

如果没有显式调用 bf.reserve 指令，第一次使用 bf.add 指令时，会创建一个默认参数的布隆过滤器。

【注意事项】

1. error_rate 设置的过小，可以提高精确度，但会占用更大的空间；
2. initial_size 设置的过小，会影响精确度；设置的过大，会占用更多的空间。当实际存储大小如果超过 initial_size，精确度也会变低。

> 对于精确度不需要特别高的场景下，error_rate 可以设置的稍大一点。设置 initial_size 时，需要尽可能的估算实际元素数量并预留一定的冗余空间以避免元素高出导致精确度降低。

# 原理
参考：https://juejin.im/post/5bc7446e5188255c791b3360

# 应用场景

1. 爬虫应用的 URL 去重
2. 邮箱系统的垃圾邮箱过滤
3. 敏感词屏蔽
4. ......
