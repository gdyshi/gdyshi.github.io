---
layout: post
title:  "GitHub 系列————actions"
categories: GitHub
tags:  GitHub 个人博客
---

* content
{:toc}
 

## 1. Actions介绍
使用paho.mqtt.embedded-c库编译的应用程序在linux环境下监听topic时，过段时间会概率性地发生连接失败并重连的现象。具体表现为会打印`yield -1`

```
void loop() {
    static int count=0;
    int ret=MQTTYield(&c, 1000);
    if (SUCCESS != ret) {
        LOGI("yield:%d",ret);
        if(0 == count%reconnect_delay){
            reconnect_count++;
            if(10<reconnect_count){
                reconnect_delay=60;
            }
            count=0;
            connect_mqtt();
        }
        count++;
    } else{
        reconnect_delay=10;
        reconnect_count=0;
        count=0;
    }
}
```

---

## 2. 问题定位

### 2.1. 确定mqtt服务端是否正常

出现问题首先要缩小问题搜索范围，然后在小范围内定位问题。网络相关的首先怀疑是否是网络问题导致的，因些我首先对网络问题进行排查。具体排查方法为：

​	使用另外的库 `mosquitto_sub` 监听同一MQTT服务的同一topic，在相同情况下`mosquitto_sub`没有问题，可以初步判定MQTT服务本身没有问题。

### 2.2. 确定是库调用的问题还是库本身的问题

确定网络没有问题后，应该可以确定是是我自己写的应用程序的问题，我自己写的应用程序也有两部分：自己写的代码和库代码。先确定是自己写的代码有问题还是库代码有问题，比较好的方式就是直接使用官方例程代码看是否有问题。如果官方例程也有问题就说明是库的问题，否则就是自己写的代码有问题。

​	在paho.mqtt.embedded-c自带监听的sample上加入失败打印，然后监听同一MQTT服务的同一topic，发现也会打印 `connect err: -1`。说明问题是出自paho.mqtt.embedded-c库

```
int main(int argc, char** argv)
{
	...
	while (!toStop)
	{
		int ret= MQTTYield(&c, 1000);	
		if (SUCCESS != ret) {
			printf("connect err:%d\n",ret);
    	}
	}
	...
}
```



### 2.3. 跟踪查找原因

#### 2.3.1. 跟踪代码

1. `MQTTYield`返回`-1`的原因只能是`cycle`返回值小于0

2. `cycle`返回值小于0有三种情况

    1. 收到PUBLISH包
    2. 收到PUBREL包
    3. 收到未识别包
    4. `keepalive`失败

3. 分别在以上4种情况加打印，然后复现问题。发现是`keepalive`失败导致错误

4. `keepalive`失败的原因只能是距离上次发送和接收都超过心跳时间，且未收到心跳回复包

正常的逻辑是不会出现这种情况的，现在问题出现的机制跟时序有关了，于是继续针对时间和`c->ping_outstanding`的操作加打印。发现了问题：出现问题时调用`keepalive`发送心跳包后没有等待`c->ping_outstanding`复位，就马上又开始了下一次的`keepalive`调用。

但是是什么机制导致`keepalive`连续调两次呢？于是又在各个可能的地方加入打印。。。

如此增加打印-分析 操作了很久，始终无法定位到原因。因为此问题本身就是概率性，又需要同时跟踪3个函数的调用时刻，并记录邻近几次调用的时间，打印就显得很有局限了。不得已我只能使用了不太熟悉的GDB

#### 2.3.2. GDB跟踪

首先对库代码做一些修改，加入保存`loop`、`MQTTYield`、`keepalive`三个函数的上次调用时间和本次调用时间，然后加入`-g`编译选项，最后运行时在`keepalive`函数的`rc = FAILURE;`异常处打断点，出现问题后，分别查看三个函数的上次调用时间和本次调用时间

```
Breakpoint 1, keepalive (c=0x6088e0 <c>)
    at /mnt/d/svn/ai_face/trunk/tcp_nat/nat_core/lib/paho.mqtt.embedded-c/MQTTClient-C/src/MQTTClient.c:224
224                 rc = FAILURE; /* PINGRESP not received in keepalive interval */
(gdb) p last
$1 = {tv_sec = 1586170992, tv_usec = 23165}
(gdb) p now
$2 = {tv_sec = 1586170992, tv_usec = 24068}
(gdb) up
#1  0x0000000000402065 in cycle (c=0x6088e0 <c>, timer=0x7ffffffedd30)
    at /mnt/d/svn/ai_face/trunk/tcp_nat/nat_core/lib/paho.mqtt.embedded-c/MQTTClient-C/src/MQTTClient.c:331
331         if (keepalive(c) != SUCCESS) {
(gdb) up
#2  0x0000000000402117 in MQTTYield (c=0x6088e0 <c>, timeout_ms=1000)
    at /mnt/d/svn/ai_face/trunk/tcp_nat/nat_core/lib/paho.mqtt.embedded-c/MQTTClient-C/src/MQTTClient.c:358
358             if (cycle(c, &timer) < 0)
(gdb) p last
$3 = {tv_sec = 1586170991, tv_usec = 23897}
(gdb) p now
$4 = {tv_sec = 1586170992, tv_usec = 23218}
(gdb) up
#3  0x0000000000401426 in loop () at /mnt/d/svn/ai_face/trunk/tcp_nat/nat_core/common/mqtt_embedded.c:138
138         int ret=MQTTYield(&c, 1000);
(gdb) p last
$5 = {tv_sec = 1586170989, tv_usec = 22031}
(gdb) p now
$6 = {tv_sec = 1586170991, tv_usec = 23895}
(gdb) down
#2  0x0000000000402117 in MQTTYield (c=0x6088e0 <c>, timeout_ms=1000)
    at /mnt/d/svn/ai_face/trunk/tcp_nat/nat_core/lib/paho.mqtt.embedded-c/MQTTClient-C/src/MQTTClient.c:358
358             if (cycle(c, &timer) < 0)
(gdb) p timer
$7 = {end_time = {tv_sec = 1586170992, tv_usec = 23896}}
(gdb) p timeout_ms
$8 = 1000
(gdb) down
#1  0x0000000000402065 in cycle (c=0x6088e0 <c>, timer=0x7ffffffedd30)
    at /mnt/d/svn/ai_face/trunk/tcp_nat/nat_core/lib/paho.mqtt.embedded-c/MQTTClient-C/src/MQTTClient.c:331
331         if (keepalive(c) != SUCCESS) {
(gdb) p *timer
$9 = {end_time = {tv_sec = 1586170992, tv_usec = 23896}}
(gdb) p packet_type
$10 = 0
(gdb) down
#0  keepalive (c=0x6088e0 <c>)
    at /mnt/d/svn/ai_face/trunk/tcp_nat/nat_core/lib/paho.mqtt.embedded-c/MQTTClient-C/src/MQTTClient.c:224
224                 rc = FAILURE; /* PINGRESP not received in keepalive interval */
(gdb) p now
$11 = {tv_sec = 1586170992, tv_usec = 24068}
(gdb) p last
$12 = {tv_sec = 1586170992, tv_usec = 23165}
(gdb) p c->last_sent
$13 = {end_time = {tv_sec = 1586171022, tv_usec = 23215}}
(gdb) p c->last_received
$14 = {end_time = {tv_sec = 1586170991, tv_usec = 3225}}
```

把各函数调用时间整理如下表：

| 函数        | 上次调用时间     | 本次调用时间     |
| ----------- | ---------------- | ---------------- |
| `loop`      | 1586170989.22031 | 1586170991.23895 |
| `MQTTYield` | 1586170991.23897 | 1586170992.23218 |
| `keepalive` | 1586170992.23165 | 1586170992.24068 |

可以看出，`loop`函数的两次调用时间间隔1S，与期望的逻辑一致；`MQTTYield`函数的两次调用时间间隔1S，与期望的逻辑一致；`keepalive`函数的两次调用时间间隔9mS，时间非常短。为什么会出现这种情况（`keepalive`多调用了一次）呢？

在函数`MQTTYield`中有这一句，可能出现多次调用`cycle`，`cycle`会调用`keepalive`

```
int MQTTYield(MQTTClient* c, int timeout_ms)
{
    ...
    do
    {
        if (cycle(c, &timer) < 0)
        {
            rc = FAILURE;
            break;
        }
  	} while (!TimerIsExpired(&timer));
	...
}
```

继续跟踪`cycle`中调用的`readPacket`方法，发现读socket操作采用了超时阻塞机制。

```
int linux_read(Network* n, unsigned char* buffer, int len, int timeout_ms)
{
	struct timeval interval = {timeout_ms / 1000, (timeout_ms % 1000) * 1000};
	if (interval.tv_sec < 0 || (interval.tv_sec == 0 && interval.tv_usec <= 0))
	{
		interval.tv_sec = 0;
		interval.tv_usec = 100;
	}

	setsockopt(n->my_socket, SOL_SOCKET, SO_RCVTIMEO, (char *)&interval, sizeof(struct timeval));

	int bytes = 0;
	while (bytes < len)
	{
		int rc = recv(n->my_socket, &buffer[bytes], (size_t)(len - bytes), 0);
		...
	}
	return bytes;
}

```

经过一番冥思苦想，嗯。。。问题可能就出在这里。

`MQTTYield`中的while是一种超时机制，`linux_read`中的socket读也是一种超时机制，但这两种超时的时刻并没有完全的同步。这样就存在一种极端情况：linux_read超时已到，但MQTTYield超时还示到，这时两者会相差很短的时间（这里抓到的是9mS）。MQTTYield会马上启动下一次cycle，而上一次cycle恰好刚刚发送了心跳包，处于等待1S内心跳回复中。但因为突然多调了一次cycle，keepalive认为在等待1S内心跳回复时间内没有收到心跳回复。

## 3. 解决方案

1. 去掉MQTTYield中的while循环，只用其中一个超时机制
2. 减少发送心跳的间隔为原来的一半，这样keepalive在还没有进行等待1S内心跳回复时就把`c->ping_outstanding`复位，就不会认为在等待心跳回复时间内没有收到心跳回复

后来我在paho.mqtt.embedded-c源码的issue中搜索此类问题，确实也有人发现了这个[问题](https://github.com/eclipse/paho.mqtt.embedded-c/pull/153)，维护人员也给了[解决方案](https://github.com/eclipse/paho.mqtt.embedded-c/commit/f9b9212c9896fd32b55f9ed942b7747fb015bd4a)。但是比较奇怪的是，这个问题在2018年12月已经解决了，却在2010年4月还没有合入主干版本

## 4. 总结

1. 出现问题后，先在官方源码ISSUE上搜索，可能已经有人提问并已经解决了。这样可以节省自己定位问题和解决问题的时间（不过这次也间接地提升了我自己定位问题的能力）
2. 定位问题时，我还是习惯于使用打印来查看。这种方式一般一次无法定位到准确位置，而且每次修改都需要重新编译代码，效率很低。后来没办法才使用了神器GDB，因为我对GDB不熟，所以不用GDB，这是我的思维惯性，后面要强制让自己多用GDB，用高效的工具来解决问题


## 参考
---
- [paho.mqtt.embedded-c 官方介绍](https://www.eclipse.org/paho/clients/c/embedded/)
- [paho.mqtt.embedded-c 源代码](https://github.com/eclipse/paho.mqtt.embedded-c)
