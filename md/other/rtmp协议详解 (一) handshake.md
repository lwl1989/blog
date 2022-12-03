CreateTime:2020-12-17 20:03:33.0

> 本文所有逻辑均从client出发

> 本文逻辑已通过golang实现，代码地址 [rtmp handshark](https://github.com/go-libraries/rtmp)

# rtmp协议是什么

RTMP服务器搭建可参考：Nginx与Nginx-rtmp-module搭建RTMP视频直播和点播服务器

实时流协议（Real-TimeMessaging Protocol，RTMP）是用于互联网上传输视音频数据的网络协议。本API提供了支持RTMP, RTMPT,RTMPE, RTMP RTMPS以及以上几种协议的变种(RTMPTE, RTMPTS)协议所需的大部分客户端功能以及少量的服务器功能。

RTMP是目前各种网络直播应用最核心的传输协议，也是互动直播采用最广泛的协议。

RTMP协议规定，播放一个流媒体有两个前提步骤：

	第一步，建立一个网络连接（NetConnection）；

	第二步，建立一个网络流（NetStream）。

其中，网络连接代表服务器端应用程序和客户端之间基础的连通关系。网络流代表了发送多媒体数据的通道。服务器和客户端之间只能建立一个网络连接，但是基于该连接可以创建很多网络流。播放一个RTMP协议的流媒体需要经过以下几个步骤：握手，建立连接，建立流，播放。

RTMP连接都是以握手作为开始的。建立连接阶段用于建立客户端与服务器之间的“网络连接”；建立流阶段用于建立客户端与服务器之间的“网络流”；播放阶段用于传输视音频数据。

# rtmp的握手过程

## 握手顺序
rtmp 连接tcp连接后，需要进行握手操作。

它包含三个固定大小的块：

	客户端发送的三个块命名为 C0,C1,C2；
	服务端发送的三个块命名为S0,S1,S2。

握手示意图：
```
    +-------------+                           +-------------+
    |    Client   |       TCP/IP Network      |    Server   |
    +-------------+            |              +-------------+
          |                    |                     |
    Uninitialized              |               Uninitialized
          |          C0        |                     |
          |------------------->|         C0          |
          |                    |-------------------->|
          |          C1        |                     |
          |------------------->|         S0          |
          |                    |<--------------------|
          |                    |         S1          |
     Version sent              |<--------------------|
          |          S0        |                     |
          |<-------------------|                     |
          |          S1        |                     |
          |<-------------------|                Version sent
          |                    |         C1          |
          |                    |-------------------->|
          |          C2        |                     |
          |------------------->|         S2          |
          |                    |<--------------------|
       Ack sent                |                  Ack Sent
          |          S2        |                     |
          |<-------------------|                     |
          |                    |         C2          |
          |                    |-------------------->|
     Handshake Done            |               Handshake Done
          |                    |                     |
              Pictorial Representation of Handshake
```

从客户端来看：

	是client先发送c0,c1给服务端，一定要等客户端收到了s1之后，才能发送c2。

从服务端来看：

	当接收到客户端的c0之后，服务端发送s0,s1给客户端。当接收到客户端的c1之后，发送s2给客户端。

整体看来：

	客户端要等收到S1之后才能发送C2
	客户端要等收到S2之后才能发送其他信息（控制信息和真实音视频等数据）
	服务端要等到收到C0之后发送S1
	服务端必须等到收到C1之后才能发送S2
	服务端必须等到收到C2之后才能发送其他信息（控制信息和真实音视频等数据） 
	如果每次发送一个握手chunk的话握手顺序会是这样：

## 握手包体

那么,rtmp的握手包是长什么样呢？

rtmp设计之初，0、1、2是相对的，保持一样的结构体。

### c0 s0

C0 和 S0 包由一个字节组成，下面是 C0/S0 包内的字段：

```
     0 1 2 3 4 5 6 7
    +-+-+-+-+-+-+-+-+
    |   version     |
    +-+-+-+-+-+-+-+-+
     C0 and S0 bits
```
version（1 byte）：RTMP 的版本，一般为 3(一般情况 写死即可)。

### c1 s1

C1和S1包含两部分数据：

key和digest，如下：

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                        time (4 bytes)                         |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                     version (4 bytes)                         |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                         key (764 bytes)                       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                      digest (764 bytes)                       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                              C1 and S1 bits
```
其实按照adobe 官方的，可以不用区分key和digest的步骤，只要能保证 剩下的1528个bt保持唯一就行

>Random data (1528 bytes): This field can contain any arbitrary values. Since each endpoint has to distinguish between the response to the handshake it has initiated and the handshake initiated by its peer,this data SHOULD send something sufficiently random. But there is no need for cryptographically-secure randomness, or even dynamic values.

我在adb官网看文档，然后结合redis rtmp实现，一直没理解协议中加密的东西。

> rtmp是adobe在2009公开的，但是rfc里规定的协议只能播放部分编码的视频，不能播放h264的视频流，原因就是handshake的原因。handshake分为simple handshake和complex handshake,RFC中只是simple handshake，complex handshake的方式才可以播放h264的流，而且我看到的所有源码的实现都是complex handshake的。

##### c1生成
key与digest顺序是不确定的，可能会调整为(time、version、digest、key)

key 结构：

	random-data: (offset) bytes
	key-data: 128 bytes
	random-data: (764 - offset - 128 - 4) bytes
	offset: 4 bytes

digest 结构：

	offset: 4 bytes
	random-data: (offset) bytes
	digest-data: 32 bytes
	random-data: (764 - 4 - offset - 32) bytes


生成Digest byte逻辑,从文档里获取到答案:

1. 获取偏移量，默认结构的话，是len(time)+len(version).则为8位
2. 计算偏移量后4位数的总和，并取余(728?),最后加上 偏移量和当前的4位.(取余操作是因为总数可能会超出整体大小)，此数用于填充结果（下标起始位）
3. 进行sha256计算，盐值为 官方公开的值GenuineFpKey.
4. 将计算结果填充到原本数据中，用到了步骤2中的起始位

offset: 找到offset那四个字节分别转换成数字相加，因为这个是随机数有可能超过了key的所有长度所有需要取余数。这个取余的数就是764字节的整体长度- 128字节的Key秘钥-4字节的自身长度

key: 这个就是128字节长度的二进制串，rtmp服务器在收到C1的key秘钥后，会生成S1的秘钥key。在官方及其各大博客中介绍的都是OpenSSL中的一个加密算法传入C1的key秘钥从而得到共享秘钥，把这个共享秘钥作为S1的key秘钥

digest:是通过秘钥算出来的一个32字节的二进制串（注意：我这里写的是秘钥，而不是秘钥key）。

##### s1校验

大致计算逻辑如下：

1.  获取pos（c1  1 、2）
2. 生成一个新数组，将s1 下标pos之前和s1 下标+32之后的数据合并为新数组
3.  进行sha256计算，盐值为 官方公开的值GenuineFmsKey.【此处对应】
4. 对比结果是否与中间的hash值相等（跳跃的32位）
5. 不相等的话，将偏移 (1528/2)=> 764+8=>772,重新计算
6. 第二次失败后，校验失败

## s2 c2

先去官网看定义吧：

>  The C2 and S2 packets are 1536 octets long, and nearly an echo of S1 and C1 (respectively), consisting of the following fields:
```
0 1 2 3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | time (4 bytes) |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | time2 (4 bytes) |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | random echo |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | random echo |
 | (cont) |
 | .... |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

c2的结构和s1保持一致，客户端的c2的数据是s1的copy.

s2: 前1504bytes数据随机生成(前8bytes需要注意),然后对1504bytes数据进行HMACsha256得到digest，将digest放到最后的32bytes即可。

s1校验需要用到c1的pos位，这里需要提前保存。

##### s2 校验

1. 获取c1的计算的hash
2. 通过c1计算的hash，结合GenuineFmsKey产生出一个新的digest。
3. 将s2的前（1536-32）的数据结合2的计算结果，计算出新的hash。
4. 用新的hash对比s2最后32位（即服务器返回的hash），匹配则正确




# 参考资料：

[adb rtmp文档](https://wwwimages2.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf "adb rtmp文档")

[handshark详解](https://blog.csdn.net/qq_28309121/article/details/104647011)

[srs rtmp协议变更](https://github.com/winlinvip/srs)
