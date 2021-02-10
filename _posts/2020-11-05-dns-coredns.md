---
layout: post
title: "DNS & CoreDNS"
author: "Gaius"
categories: Devops
tags: [documentation]
image: dns-coredns.jpg
---

## DNS

DNS（Domain Name System）是一个全球化的分布式数据库，用于存储域名与互联网 IP 地址的映射关系。DNS 分为两大类：权威 DNS，递归 DNS。

### 权威 DNS

权威 DNS 是特定域名记录在域名注册商处所设置的 DNS 服务器，用于特定域名本身的管理。且只对自己所拥有的域名进行域名解析，非自己的域名则拒绝访问。

### 递归 DNS

递归 DNS 又称 Local DNS，用于域名查询。递归 DNS 会迭代权威服务器返回的应答，直至最终查询到的 IP 地址，将其返回给客户端，并将请求结果缓存到本地。

### DNS 记录

A 记录： 指向 IPv4 地址。

AAAA 记录： 指向 IPv6 地址。

CNAME 记录： 指向域名。

NS 记录： 指定域名解析服务器，即负责管理域名对应 IP 记录服务器。

MX 记录： 指向邮件服务器地址。

TXT 记录： 可任意填写，可为空。

### TTL

表示解析记录在递归 DNS 中的缓存时间，单位秒。递归 DNS 得到解析记录后，会依照 TTL 进行缓存，但是有可能递归 DNS 不遵循 TTL，比如递归 DNS 觉得 TTL 太小，频繁的递归请求会耗尽递归 DNS 的资源。优化减少请求到权威 DNS 服务器的次数，尽量访问递归 DNS，需要调高 TTL 时长，让客户端尽量访问递归 DNS 缓存。但是 TTL 时长较长可能导致更改源 DNS 记录无效，因为记录值依照 TTL 规范缓存在递归 dns 中。

### DNS 解析过程

浏览器输入 www.airbnb.com 地址，则首先根据 Domain 查询 DNS 服务并给出对应 IP。完整的递归 DNS 查询流程需要 DNS 服务器从根域名 "." 服务器，顶级域名服务器 ".com"，一级域名服务器 "airbnb.com"，一级一级递归查询，直到最终找到权威服务器取得结果，并返回给客户。同时，递归服务器根据域名 TTL，缓存查询结果，便于相同域名重复查询。

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gkfq7ve54wj31300pw42n.jpg)

第一次解析时间较长因为有回源权威，第二次解析记录根据 TTL 在递归 DNS 进行缓存。
![](https://tva1.sinaimg.cn/large/0081Kckwgy1gkfqjzza6tj30qo03xjsg.jpg)

## CoreDNS

CoreDNS 是一个灵活可扩展的 DNS 服务器，可以作为 Kubernetes 集群 DNS。与 Kubernetes 一样，CoreDNS 项目由 CNCF 托管，其大多数功能都是由插件实现。

### DNS In Kubernetes

> DNS Policy: [https://godoc.org/k8s.io/api/core/v1#DNSPolicy](https://godoc.org/k8s.io/api/core/v1#DNSPolicy)

Pod dnsPolicy 为 ClusterFirst，则使用集群 DNS，而非宿主机 DNS 配置。

![](https://tva1.sinaimg.cn/large/0081Kckwly1gkfr1so3umj30iu0qdgpb.jpg)

查看 Pod 中 DNS 配置文件 /etc/resolv.conf。nameserver 即为 CoreDNS Service 对应 Virtual IP。search 为搜索列表。options ndots 表示大于特定数字，不走 search 按照原域名进行解析，否则会按照 search 列表中逐一匹配查询，如果都是 Not Found 则按照原域名进行解析。

![](https://tva1.sinaimg.cn/large/0081Kckwly1gkfquvrxxbj30h5021jrk.jpg)

CoreDNS SVC 对应 Virtual IP，即 Pod 中 DNS 配置文件 nameserver 服务器 IP。

![](https://tva1.sinaimg.cn/large/0081Kckwly1gkfr6vf7bhj30s404mabf.jpg)
