---
layout: post
title: "WWW & Root Record"
author: "Gaius"
categories: Devops
tags: [documentation]
image: domain-with-www.jpg
---

## 概念

### WWW
[万维网(World Wide Web)](https://en.wikipedia.org/wiki/World_Wide_Web)是一个透过互联网访问的，由许多互相链接的超文本组成的系统。英国科学家蒂姆·伯纳斯-李于 1989 年发明了万维网。1990 年他在瑞士 CERN 的工作期间编写了第一个网页浏览器。网页浏览器于 1991 年在 CERN 以外发行，1991 年 1 月最先向其他研究机构发行，并于 1991 年 8 月在互联网上向公众开放。

### 顶点记录 
顶点记录位于 DNS 区域的根(或顶点)中的 DNS 记录。 例如，在 DNS 区域 `airbnb.com` 中，顶点记录还具有完全限定的名称 `airbnb.com`, 即称为裸域。按照惯例，相对名称 `@` 用于表示顶点记录。

## 实践经验

### 顶点记录 & CNAME
[RFC 1034 3.6.2](http://www.faqs.org/rfcs/rfc1034.html) 给不推荐顶点记录添加 CNAME，需要直接配置 A 记录。万网的权威 DNS 服务，如果给顶点记录直接添加 CNAME 会直接报错。

主要原因是假如给 `airbnb.com` 顶点记录添加 CNAME 至 `facebook.com`，则相当于给 `airbnb.com` 添加 Alias `facebook.com`。则第一次访问 `airbnb.com` 会把 `facebook.com` 记录本机缓存，下次访问 `airbnb.com` 则直接使用本地缓存直接访问 `facebook.com`。

同时被 Alias 的还有 MX(Mail eXchange) 记录即邮箱服务记录, 也就导致给 `airbnb.com` 发送邮件相当于给 `facebook.com` 发送邮件。而反过来先发送邮件，再访问网页则会给 `airbnb.com` 发送邮件，因为本地没有缓存 CNAME。

正常访问逻辑应该是 `airbnb.com` 网页访问到 `facebook.com`, 但是发送邮件到 `airbnb.com`。当添加 CNAME 给顶点记录时会导致邮箱服务出现问题。

当然类似 AWS、Azure 以及 Aliyun 等 DNS 服务是允许给顶点记录添加 CNAME 的，是因为基于 [CloudFlare](https://blog.cloudflare.com/introducing-cname-flattening-rfc-compliant-cnames-at-a-domains-root) 方式。CloudFlare 原理为递归解析配置 CNAME 并转换成 A 记录。

正常顶点记录与 WWW DNS 配置方式为:

```text
airbnb.com        A   52.7.164.128
www.airbnb.com  CNAME airbnb.com
```

### 浏览器访问特征
浏览器访问时会根据具体输入进行自动补全，查询不同的 DNS 记录。

```text
https://airbnb.com      => airbnb.com
https://www.airbnb.com  => www.airbnb.com
airbnb.com              => www.airbnb.com
www.airbnb.com          => www.airbnb.com
```

### TLS 证书
WWW 解析记录可以看作顶点记录的一个子域，类似 `zh.airbnb.com` 与 `www.airbnb.com` 在 DNS 中是等价的。所以 `zh.airbnb.com` 和 `www.airbnb.com` 使用的 TLS 证书是 `*.alipay.com`, 而 `airbnb.com` 使用的 TLS 证书是 `alipay.com`

```text
www.airbnb.com      => *.alipay.com
airbnb.com          => alipay.com
```
