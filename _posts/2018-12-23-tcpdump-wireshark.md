---
layout: post
title: "Tcpdump & Wireshark"
author: "Gaius"
categories: Devops
tags: [documentation]
image: tcpdump-wireshark.png
---

## TCP 标志位(TCP Flags)
> Transmission Control Protocol: [https://en.wikipedia.org/wiki/Transmission_Control_Protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gkelu9itx8j30vc0dgjtn.jpg)

TCP 首部中 TCP Flags 位于第 14 个字节, 包含 8 个比特位。主要用作操控 TCP 状态机, 8 个比特位都表示特定功能, 分别为:

CWR & ECE: 用来配合做 congestion control，一般情况下和应用层关系不大。发送方的包 ECE 为 0 时，表示出现了 congestion。接收方回的包里 CWR 为 1 时，表示收到 congestion 信息并做了处理。

URG: 表示 TCP 包的紧急指针域有效。用来保证 TCP 连接不被中断，并且督促中间层设备要尽快处理这些数据。

ACK: 表示应答域有效。有两个取值 0 和 1，为1的时候表示应答域有效，反之为0。

PSH: 表示 Push 操作。所谓 Push 操作就是指在数据包到达接收端以后，立即传送给应用程序， 而不是在缓冲区中排队。

RST: 表示连接复位请求。用来复位那些产生错误的连接，也被用来拒绝错误和非法的数据包。

SYN: 表示同步序号，用来建立连接。SYN 标志位和 ACK 标志位搭配使用，当连接请求的时候 SYN=1，ACK=0。连接被响应的时候 SYN=1，ACK=1。

FIN: 表示发送端已经达到数据末尾。也就是说双方的数据传送完成，没有数据可以传送了，发送 FIN 标志位的 TCP 数据包后，连接将被断开。

## 连接复位(RST)

TCP Flags 用 RST 表示复位，用来表示异常关闭连接。发送 RST 包关闭连接时，不必等缓冲区的包都发出去，直接就丢弃缓存区的包发送 RST 包。而接收端收到 RST 包后，也不必发送 ACK 包来确认。主要有以下四种情况可能导致 RST:

- 端口未打开
- Request 请求超时
- Socket 提前关闭
- 已经关闭 Socket 上收到数据

## 线上排查过程

排查请求异常断连, 应用日志显示为 Read ECONNRESET 其实日志内容已经告诉了问题原因, ECONNRESET 即收到了 TCP RST 报文。
```bash
Error: read ECONNRESET at TCP.onStreamRead (internal/stream_base_commons.js:200:27) {
  errno: 'ECONNRESET',
  code: 'ECONNRESET',
  syscall: 'read',
  name: 'ResponseError',
  data: undefined,
  path: '/mgw.htm',
  status: -1,
  headers: {},
  res: {
    status: -1,
    statusCode: -1,
    statusMessage: null,
    headers: {},
    size: 0,
    aborted: false,
    rt: 90360,
    keepAliveSocket: false,
    data: undefined,
    requestUrls: [ 'http://google.com' ],
    timing: null,
    remoteAddress: '108.177.8.139',
    remotePort: 80,
    socketHandledRequests: 1,
    socketHandledResponses: 0
  }
}
```

登陆 CentOS or Fedora 服务器安装 tcpdump，正常情况 Linux & macOS 默认安装。

```bash
sudo yum install tcpdump
```

服务器内用 tcpdump 抓包, 数据倒入 out.pcap 文件内。

```bash
tcpdump -i any -w out.pcap
```

或者根据 IP 进行抓包。
```bash
tcpdump -i any host 8.8.8.8 -w out.pcap
```

out.pcap 文件导出, 权限限制情况下 scp 不可取。所以用 OSS 当媒介上传, 安装阿里云 OSS Client。
```bash
wget http://gosspublic.alicdn.com/ossutil/1.6.18/ossutil64
chmod 755 ossutil64
```

生成 OSS 相关配置文件。
```bash
./ossutil64 config -e oss-ap-southeast-1.aliyuncs.com -i ak-id -k ak-secret  -L CH -c ./myconfig
```

上传 .pcap 文件至 OSS。
```bash
./ossutil64 cp out.pcap oss://oss-ap-southeast-1.aliyuncs.com/out.pcap --config-file ./myconfig
```

out.pcap 在 Wireshark 中打开进行分析。
![](https://tva1.sinaimg.cn/large/0081Kckwly1gkemwv7qf7j31d90u0npf.jpg)

通过 tcp.flags.reset == 1 过滤出 RST 异常数据包。
![](https://tva1.sinaimg.cn/large/0081Kckwly1gkemyjprk4j31dh0u0446.jpg)

根据 IP 以及 Port 端口号 ip.src eq 121.0.26.95 or ip.dst eq 121.0.26.95 and tcp.port == 49688 过滤出，整体 TCP 传输报文。
![](https://tva1.sinaimg.cn/large/0081Kckwly1gken04741vj324g0lodu9.jpg)

数据报文进行分析, 首先进行 TCP 三次握手。
![](https://tva1.sinaimg.cn/large/0081Kckwly1gken1bbkytj324k0lc7j0.jpg)

报文解析客户端到服务端数据报文, 2047 为帧号。
![](https://tva1.sinaimg.cn/large/0081Kckwly1gken7kxu1vj31d90u0npd.jpg)

2048 帧，服务端的返回报文，每个报文都有对应的响应报文。
![](https://tva1.sinaimg.cn/large/0081Kckwly1gken392gbkj31d90u0gso.jpg)

2057 帧后客户端 3s 没有返回，服务端超时，发送 RST 报文中止了链接。
![](https://tva1.sinaimg.cn/large/0081Kckwly1gken40n91qj31d90u0e81.jpg)
![](https://tva1.sinaimg.cn/large/0081Kckwly1gken3y5k8mj31d90u0npd.jpg)

原因为服务端发送 RST 报文中止请求导致。正常情况下中止是通过服务端发送 FIN 四次挥手结束的。
![](https://tva1.sinaimg.cn/large/0081Kckwly1gken6d62wwj31df0u0180.jpg)

