
Redis 使用了一种“浪费流量”的文本协议，优势在于实现过程简单，解析性能极好。它将传输的结构数据分为 5 种最小单元类型，单元结束时统一加上换行符号 `\r\n`

1. 单行字符串以 `"+"` 符号开头。
2. 多行字符串以 `"$"` 符号开头，后跟字符串长度。
3. 整数值以 `":"` 号开头，后跟整数的字符串形式。
4. 错误类型以 `"-"` 符号开头。
5. 数组以 `"*"` 符号开头，后跟数组的长度。
<!-- more -->
【单行字符串】

    # 单行字符串 hello world 的序列化表示
    +hello world\r\n

【多行字符串】

    $11\r\nhello world\r\n

【整数】

    # 1024 的序列化表示
    :1024\r\n

【错误】

    -WRONGTYPE.......\r\n

【数组】

    [1,2,3]
    *3\r\n:1\r\n:2\r\n:3\r\n

【null】

    # null 需要用多行字符串表示，不过长度设置为 -1
    $-1\r\n

【空串】

    $\r\n\r\n

## 客户端向服务端的传输指令的过程
客户端向服务端传输指令只有一种格式，就是多行字符串数组。一个 set 指令 set author codehole 会被序列化成下面的字符串：

    # set author codehole
    *3\r\n$3\r\nset\r\n$6\r\nauthor\r\n$8\r\ncodehole\r\n
    # 控制台输出时，将会是很容易阅读的一种方式：
    *3
    $3
    set
    $6
    author
    $8
    codehole

## 服务端给客户端返回结果的过程

单行字符串响应：

    > set author codehole
    OK # 控制台输出
    --------------------------
    # 序列化文本，单行响应
    +OK

错误响应：

    > incr author
    （error）ERR value is not an integer or out pf range
    --------------------------
    # 响应的序列化文本
    -ERR value is not an integer or out pf range

整数响应：

    > incr books
    （integer）1
    --------------------------
    # 响应的序列化文本，整数响应
    :1

多行字符串响应：

    > get author
    "codehole"
    --------------------------
    # 响应的序列化文本
    $8\r\ncodehole\r\n

数组响应：

    > hset info name harry
    （integer）1
    > hset ingo age 22
    （integer）1
    > hgetall info
    1）"name"
    2）"harry"
    3）"age"
    4）"22"
    --------------------------
    # 响应的序列化文本
    *4\r\n$4\r\nname\r\n$5\r\nharry\r\n$3\r\nage\r\n$2\r\n22\r\n
