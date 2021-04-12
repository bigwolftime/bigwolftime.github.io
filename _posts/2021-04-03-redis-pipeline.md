---
layout: post
title: Redis pipeline
date: 2021-04-03
Author: bwt
categories: Redis
tags: [Redis, Jedis, pipeline]
comments: true
toc: true
---

> 文中使用 Jedis 进行数据交互, 版本为 2.6.1

#### 一. 使用场景

我们在使用 Redis 的过程中, 不免要进行批量的数据操作, 例如: 根据一个 uidList 查询出每个用户的信息, 假设用户信息的缓存使用 Key-Value 
结构存储, 在不使用 pipeline 的情况下, 我们的代码可能是这样的(以 Jedis 组件为例):

```java
public List<User> getUserInfoByUids(List<Long> uidList) {
    List<String> userList = new ArrayList<>();

    for (long uid : uidList) {
        userList.add(jedis.get("user_info" + uid));
    }
    
    return userList;
}
```

使用这种方式调用, 其流程如图:

![批量操作不使用 pipeline 的交互](https://zonheng.net/redis_command_one_by_one.png)

这样做有以下问题:

1. 每次 send command 都代表一次网络请求, 如果业务量稍大, 就会占用大量的网络带宽;
2. Redis Server 每接收并处理一个请求, 就需要使用系统调用中的 read() / write(), 这意味着应用程序上下文会从用户态陷入到内核态, 这也是
   一项性能损耗.

下面代码使用 Redis pipeline 执行批量 set 操作(只是举例说明 pipeline 的使用, 此处也可以使用 `mset` 命令). 

```java
public void saveData(Map<String,String> map) {
    List<String> userList = new ArrayList<>();
    Pipeline pipeline = jedis.pipelined();
    
    for (Map.Entry<String, String> entry : map) {
        pipeline.set("key_" + entry.getKey(), entry.getValue());
    }
    
    pipeline.sync();
}
```

在将所有的命令都准备完成后, 调用 `pipeline.sync() / pipleline.syncAndReturnAll()` 将一批命令一次性发给 Redis Server, 待 Redis 
Server 执行完毕后, 将执行结果一次性地返回给客户端.

**在使用 pipeline 的情况下, 可将多次往返的网络请求降低到一次往返; 处理这批命令时也只需一次系统调用(read() / write()), 可以有效地提升
系统性能.**

**注意事项**

在使用 pipeline 时, 特别要注意控制命令的数量, 如果一次组装 pipeline 的数据量过大, 会造成以下问题:

1. Redis Server 要执行的命令较多, 增加客户端等待时间; 
2. 网络带宽的瞬时占用增大, 甚至出现网络阻塞;
3. Redis Server 执行完毕后, 答复数据会在内存中按顺序存储, 待剩余命令执行完毕后返回, 此逻辑可能会造成内存被大量占用.

所以当要查询的 key 数据量较大时, 可以想办法将数据分割成 n 个小份, 分批通过 pipeline 查询. 例如：有一个数据量为 10w 的 uidList, 要
根据每个 uid 查询出用户信息, 可尝试这样写:

```java
public List<String> getUserListByUid(List<Long> uidList) {
    List<String> dataList = new ArrayList<>();
    
    // 使用 google common 包，将 uidList 分割成多个子 list, 每个子 list 最多 500 元素
    for (List<Long> subList : Lists.partition(uidList, 500)) {
        
        // 遍历每个子 list, 查询用户信息
        List<Response<String>> respList = new ArrayList<>();
        for (long uid : subList) {
            respList.add(pipeline.get("user_info_" + uid));
        }
        
        // 同步 pipeline
        pipeline.sync();
        
        // 将查询到的结果装入到 dataList 中
        dataList.addAll(respList.stream().filter(Objects::nonNull).map(Response::get).collect(Collectors.toList()));
    }
     
     return dataList;
}
```

#### 二. Jedis pipeline 代码实现

Jedis pipeline 的各种命令, 最终都会调用到 `Protocol#sendCommand(...)`, 逻辑如下:

```java
private static void sendCommand(final RedisOutputStream os, final byte[] command, final byte[]... args) {
    // Command 是一个枚举, 用来标记什么命令, 其可选值举例: PING SET GET EXISTS...
    // args 是参数数组, args[0] 一般代表要操作的 key
	try {
	    os.write(ASTERISK_BYTE);
	    os.writeIntCrLf(args.length + 1);
	    os.write(DOLLAR_BYTE);
	    os.writeIntCrLf(command.length);
	    os.write(command);
	    os.writeCrLf();

	    for (final byte[] arg : args) {
           os.write(DOLLAR_BYTE);
           os.writeIntCrLf(arg.length);
           os.write(arg);
           os.writeCrLf();
	    }
	} catch (IOException e) {
	    throw new JedisConnectionException(e);
	}
}
```

其主要逻辑是维护一个 RedisOutputSteam 的 byte 数组(变量 os), 用来存储命令(get, set...)以及参数等信息.

再看 `RedisOutputStream#write()` 和 `RedisOutputStream#flushBuffer()` 方法:

```java
public void write(final byte b) throws IOException {
	if (count == buf.length) {
	    flushBuffer();
	}
	buf[count++] = b;
}


private void flushBuffer() throws IOException {
     if (count > 0) {
        out.write(buf, 0, count);
        count = 0;
     }
}
```

首先判断当前的 RedisOutputStream 是否达到了 8192byte(8192 通过 RedisOutputStream 的构造方法初始化), 如果已达到则执行
`flushBuffer()`, 将这一批次的命令发送给 Redis Server.

最后通过调用 `pipeline.sync() / pipeline.syncAndReturnAll()` 方法, 此操作将读取 RedisInputStream 的响应数据并同步, 最后关闭
pipeline.

通过上面的分析可以看到, **同步 pipeline 有两种办法: 缓冲区达到阈值(8192); 手动调用了 sync. 所以在日常开发中, 不要忘记调用 sync, 否则
命令会暂存在 RedisOutputStream, 并不会真正地发送给 Redis Server, 若此时服务故障或重启, 则会造成数据丢失.**

#### 三. 参考

* [Using pipelining to speedup Redis queries](https://redis.io/topics/pipelining)
* [巧用 Redis pipeline 命令，解决真实的生产问题](https://mp.weixin.qq.com/s/54n1Q3_Zvyxr9Sj2Fqzhew)
