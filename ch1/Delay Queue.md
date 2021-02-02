# 异步消息队列
使用 redis 的 list 数据结构可以实现简单的消息队列效果。redis 消息队列没有 ask 机制，如果对消息的可靠性有严格要求则应该使用专业的消息队列中间件如 rabbitmq、kafka。

要使用 redis 的消息队列可以通过搭配 `rpush、lpop` 或 `lpush、rpop` 命令实现。

生产者通过 rpush、lpush 推送消息到 list 中，消费者不断轮询 key，通过 rpop、lpop 获取消息。
<!-- more -->
# 队列为空带来的问题
客户端通过队列的 pop 指令获取消息，处理消息，再获取再处理，如此循环。如果队列空了，客户端就会一直进行无效的 pop，这样会拉高服务器的 cpu 消耗如何避免呢？

# 阻塞读
使用 blpop/brpop 指令代替 lpop/rpop 指令。b 是 blocking 的意思，阻塞读在没有数据的时候，会进入到休眠状态，当一有数据到来会立刻立刻醒过来。

需要注意的是，线程一直阻塞着等来消息过来可能会被当作是闲置线程，闲置过久服务器一般会主动断开连接以减少闲置资源占用。此时 blpop/brpop 会抛出异常，所以编写客户端消费者时需要小心，当捕获此异常时，需要重试。

# 延迟队列
使用 redis 的 zset 可以实现延迟队列。消息体存为 value，消息的处理时间作为 score，使用多个线程轮询 zset 获取即将过期的任务进行处理。多个线程是为了保障可用性，万一一个线程挂掉了其他线程可以继续处理。

有了多个线程，就需要确保并发争抢任务的问题，确保任务不会被多次执行。

# 练习

延迟队列：[/project/1-4/src/main/java/xyz/yuzh/learning/redis/service/RedisDelayingQueue.java](https://github.com/yuzh233/redis-learning/blob/master/project/1-4/src/main/java/xyz/yuzh/learning/redis/service/RedisDelayingQueue.java)
