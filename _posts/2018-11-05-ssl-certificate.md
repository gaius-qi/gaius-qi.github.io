---
layout: post
title: "SSL 证书实践"
author: "Gaius"
categories: Devops
tags: [documentation]
image: ssl-certificate.png
---

SSL 证书实践过程中，避免不了遇到各种问题，本文主要介绍实践过程中相关经验。

## 概念
## SSL、TLS & HTTPS
SSL: 指安全套接字层，简而言之，它是一项标准技术，可确保互联网连接安全，保护两个系统之间发送的任何敏感数据。

TLS: 传输层安全是更为安全的升级版 SSL。

HTTPS: 基于TLS/SSL的安全套接字上的的应用层协议，除了传输层进行了加密外，其它与常规HTTP协议基本保持一致。

## TLS 版本
可以通过 [ssl-labs](https://www.ssllabs.com/ssltest/analyze.html) 来检查, 域名对应 TLS 版本信息。

![](https://tva1.sinaimg.cn/large/0081Kckwly1gkeexgqk82j31gc0dgta4.jpg)

NGINX: 配置 TLS 版本参考 [ssl-protocols](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_protocols)。

NGINX Ingress: 默认使用兼容性好且安全性高的 TLS v1.2, 配置具体版本参考 [configmap-ssl-protocols](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#ssl-protocols)。

## 证书
### 证书链分级
完整的证书内容一般分为3级，服务端证书-中间证书-根证书，即 End-user Certificates， Intermediates Certificates 和 Root Certificates。
- End-user Certificates：用来加密传输数据的公钥的证书，是https中使用的证书。开发者把证书部署在 marmot-cloud.com 服务器上。
- Intermediates Certificates：CA用来认证公钥持有者身份的证书，即确认 https 使用的 end-user 证书是属于 marmot-cloud.com 的证书。
- Root Certificates：用来认证 intermediates 证书是合法证书的证书。

### 证书链修复
证书链不完整常会报错如下:
```text
the certificate is not trusted in all web browsers. you may need to install an intermediate/chain certificate to link it to a trusted root certificate.
```
引起该问题原因有很多, 比如老版本客户端系统 Root Certificates 未默认集成导致证书链不全。可以通过提供 End-user Certificates 基于 [my-ssl](https://myssl.com/chain_download.html) 来修复完成证书链。

### 证书链查询
```bash
openssl s_client -servername airbnb.com -connect airbnb.com:443 -showcerts
```

查询结果显示, 0 级为 End-user Certificates(airbnb.com), 1 级为 Intermediates Certificates, 2 级为 Root Certificates。
```text
CONNECTED(00000006)
depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
verify return:1
depth=1 C = US, O = DigiCert Inc, CN = DigiCert SHA2 Secure Server CA
verify return:1
depth=0 C = US, ST = California, L = San Francisco, O = "Airbnb, Inc.", CN = airbnb.com
verify return:1
---
Certificate chain
 0 s:/C=US/ST=California/L=San Francisco/O=Airbnb, Inc./CN=airbnb.com
   i:/C=US/O=DigiCert Inc/CN=DigiCert SHA2 Secure Server CA
-----BEGIN CERTIFICATE-----
MIIIvjCCB6agAwIBAgIQA/y0YGMRY6a9g87hGSCsxzANBgkqhkiG9w0BAQsFADBN
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMScwJQYDVQQDEx5E
aWdpQ2VydCBTSEEyIFNlY3VyZSBTZXJ2ZXIgQ0EwHhcNMjAwNjA1MDAwMDAwWhcN
MjIwMjA3MTIwMDAwWjBmMQswCQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5p
YTEWMBQGA1UEBxMNU2FuIEZyYW5jaXNjbzEVMBMGA1UEChMMQWlyYm5iLCBJbmMu
MRMwEQYDVQQDEwphaXJibmIuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIB
CgKCAQEAysTmtbp49OE4RrOiV8ctfB8moVQI99IOKIuLojnv55lt2kKDSkSxhZX1
zgDZtMKN6ziYjz2+pkqpEtdsbuNxcFaH9Aadc2bKH6OnuXkVoGSvPGKIv+aR7Wu6
Ylg01W6liBF0+PGEVb9Ea9gbKnzc+8gy6RVpQ+e19/NVBui6S0F9k290B/GdCjGZ
Yk7f6rZCLCBEQ6gPsLtXroIzAX+1fKyiUOHf3ljzAMqPfyyPwVmDMXKGhXDcKopR
qccsdEjPEiOsVpPVRX2SKdyou2D95HSTQsErOGhJEbU+ZvNaBqeubUerIxz0/07n
0mShWRNQ/jSnSim0Ba6VW2NoOsvkqwIDAQABo4IFfzCCBXswHwYDVR0jBBgwFoAU
D4BhHIIxYdUvKOeNRji0LOHG2eIwHQYDVR0OBBYEFN2jfgK3oODTwv+vLe8jRshe
+KhbMIICPgYDVR0RBIICNTCCAjGCCmFpcmJuYi5jb22CDWFpcmJuYi50cmF2ZWyC
CWFpcmJuYi5ka4IJYWlyYm5iLmZygglhaXJibmIuZmmCCWFpcmJuYi5kZYIJYWly
Ym5iLm14ggphYmIudHJhdmVsggphaXJibmIuY2F0gglhaXJibmIuanCCCWFpcmJu
Yi5pdIIJYWlyYm5iLmlzgglhaXJibmIucGyCCWFpcmJuYi5ncoIKYWlyYm5iLm9y
Z4IIdmFtby5jb22CCWFpcmJuYi5ydYIJYWlyYm5iLmJlgglhaXJibmIubmyCCWFp
cmJuYi5hdIIJYWlyYm5iLnB0gglhaXJibmIuY2iCCWFpcmJuYi5pZYIJYWlyYm5i
LmNhgglhaXJibmIuc2WCCWFpcmJuYi5jeoINYWNjb21hYmxlLmNvbYIJYWlyYm5i
Lm5vgglhaXJibmIuZXOCCWFpcmJuYi5odYINYWlyYm5iLmNvbS5zZ4IMYWlyYm5i
LmNvLm56ggxhaXJibmIuY28udWuCDWFpcmJuYi5jb20udm6CDGFpcmJuYi5jby5p
ZIINYWlyYm5iLmNvbS51YYINYWlyYm5iLmNvbS5icoIMYWlyYm5iLmNvLmtygg1h
aXJibmIuY29tLm15gg1haXJibmIuY29tLmF1gg1haXJibmIuY29tLmhrgg1haXJi
bmIuY29tLnR3gg1haXJibmIuY29tLmhygg1haXJibmIuY29tLnRyggxhaXJibmIu
Y28uaWwwDgYDVR0PAQH/BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEF
BQcDAjBrBgNVHR8EZDBiMC+gLaArhilodHRwOi8vY3JsMy5kaWdpY2VydC5jb20v
c3NjYS1zaGEyLWc2LmNybDAvoC2gK4YpaHR0cDovL2NybDQuZGlnaWNlcnQuY29t
L3NzY2Etc2hhMi1nNi5jcmwwTAYDVR0gBEUwQzA3BglghkgBhv1sAQEwKjAoBggr
BgEFBQcCARYcaHR0cHM6Ly93d3cuZGlnaWNlcnQuY29tL0NQUzAIBgZngQwBAgIw
fAYIKwYBBQUHAQEEcDBuMCQGCCsGAQUFBzABhhhodHRwOi8vb2NzcC5kaWdpY2Vy
dC5jb20wRgYIKwYBBQUHMAKGOmh0dHA6Ly9jYWNlcnRzLmRpZ2ljZXJ0LmNvbS9E
aWdpQ2VydFNIQTJTZWN1cmVTZXJ2ZXJDQS5jcnQwDAYDVR0TAQH/BAIwADCCAX8G
CisGAQQB1nkCBAIEggFvBIIBawFpAHYAKXm+8J45OSHwVnOfY6V35b5XfZxgCvj5
TV0mXCVdx4QAAAFyhhhGAQAABAMARzBFAiEAupludhrxIbp4FBRTl3Wxh/688g2a
FcE6TMZ6D1zlCFICIBTi/C5AIKjfM7hosedQgwScfNgIzluEZy/DOBlzvs0yAHcA
QcjKsd8iRkoQxqE6CUKHXk4xixsD6+tLx2jwkGKWBvYAAAFyhhhFugAABAMASDBG
AiEAgP1ZY3UMrFvWVKP5XhMhfXQefbb5KCnfyJCWaISd9dkCIQC3nddMiCmn9c7e
4rwAFASvJUDhqeWOeP20HnT1XCQ3qAB2AEalVet1+pEgMLWiiWn0830RLEF0vv1J
uIWr8vxw/m1HAAABcoYYRkcAAAQDAEcwRQIgfdUJZTJCxSl87rZWarSqLTZwSWCU
ucRNiC7MYGXvHL8CIQCGt12tJL1FYwn3WqjxoW7G9SuqQrHGNg/xcWgiURPHfjAN
BgkqhkiG9w0BAQsFAAOCAQEAi81YDzYL1LNb1TPkqRYFMvFrKT5RFADKNwjXg/vD
xHKVlshhTQ2aXqy70dry0nBUCQWDdLUKJBkA8tG3uRq9v3X1FDZjR2x+lDcCpdm3
y5b1RDq8l57hl7ChbeTfi4/QpE7M2ncmcX+em+sq3osS12GswCZVURxEk5cKRn/z
Pn5BTz09o/Nse277Noxk0T3vNH1meXbYlBQ6vXc/qWQWzO6NtlehuAhPZgHypqqg
4jkHJxlzWZIjQZdP9afqpWwbenvCfzjnlSDmUBfwOXtzBzwKepkO0j3/bDO5ayD1
y5vqxhMovE/t+KPT1Rp3F1+982HSCQ7wuoCftlfxdesYJg==
-----END CERTIFICATE-----
 1 s:/C=US/O=DigiCert Inc/CN=DigiCert SHA2 Secure Server CA
   i:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root CA
-----BEGIN CERTIFICATE-----
MIIElDCCA3ygAwIBAgIQAf2j627KdciIQ4tyS8+8kTANBgkqhkiG9w0BAQsFADBh
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD
QTAeFw0xMzAzMDgxMjAwMDBaFw0yMzAzMDgxMjAwMDBaME0xCzAJBgNVBAYTAlVT
MRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxJzAlBgNVBAMTHkRpZ2lDZXJ0IFNIQTIg
U2VjdXJlIFNlcnZlciBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
ANyuWJBNwcQwFZA1W248ghX1LFy949v/cUP6ZCWA1O4Yok3wZtAKc24RmDYXZK83
nf36QYSvx6+M/hpzTc8zl5CilodTgyu5pnVILR1WN3vaMTIa16yrBvSqXUu3R0bd
KpPDkC55gIDvEwRqFDu1m5K+wgdlTvza/P96rtxcflUxDOg5B6TXvi/TC2rSsd9f
/ld0Uzs1gN2ujkSYs58O09rg1/RrKatEp0tYhG2SS4HD2nOLEpdIkARFdRrdNzGX
kujNVA075ME/OV4uuPNcfhCOhkEAjUVmR7ChZc6gqikJTvOX6+guqw9ypzAO+sf0
/RR3w6RbKFfCs/mC/bdFWJsCAwEAAaOCAVowggFWMBIGA1UdEwEB/wQIMAYBAf8C
AQAwDgYDVR0PAQH/BAQDAgGGMDQGCCsGAQUFBwEBBCgwJjAkBggrBgEFBQcwAYYY
aHR0cDovL29jc3AuZGlnaWNlcnQuY29tMHsGA1UdHwR0MHIwN6A1oDOGMWh0dHA6
Ly9jcmwzLmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydEdsb2JhbFJvb3RDQS5jcmwwN6A1
oDOGMWh0dHA6Ly9jcmw0LmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydEdsb2JhbFJvb3RD
QS5jcmwwPQYDVR0gBDYwNDAyBgRVHSAAMCowKAYIKwYBBQUHAgEWHGh0dHBzOi8v
d3d3LmRpZ2ljZXJ0LmNvbS9DUFMwHQYDVR0OBBYEFA+AYRyCMWHVLyjnjUY4tCzh
xtniMB8GA1UdIwQYMBaAFAPeUDVW0Uy7ZvCj4hsbw5eyPdFVMA0GCSqGSIb3DQEB
CwUAA4IBAQAjPt9L0jFCpbZ+QlwaRMxp0Wi0XUvgBCFsS+JtzLHgl4+mUwnNqipl
5TlPHoOlblyYoiQm5vuh7ZPHLgLGTUq/sELfeNqzqPlt/yGFUzZgTHbO7Djc1lGA
8MXW5dRNJ2Srm8c+cftIl7gzbckTB+6WohsYFfZcTEDts8Ls/3HB40f/1LkAtDdC
2iDJ6m6K7hQGrn2iWZiIqBtvLfTyyRRfJs8sjX7tN8Cp1Tm5gr8ZDOo0rwAhaPit
c+LJMto4JQtV05od8GiG7S5BNO98pVAdvzr508EIDObtHopYJeS4d60tbvVS3bR0
j6tJLp07kzQoH3jOlOrHvdPJbRzeXDLz
-----END CERTIFICATE-----
 2 s:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root CA
   i:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root CA
-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIQCDvgVpBCRrGhdWrJWZHHSjANBgkqhkiG9w0BAQUFADBh
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD
QTAeFw0wNjExMTAwMDAwMDBaFw0zMTExMTAwMDAwMDBaMGExCzAJBgNVBAYTAlVT
MRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5j
b20xIDAeBgNVBAMTF0RpZ2lDZXJ0IEdsb2JhbCBSb290IENBMIIBIjANBgkqhkiG
9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4jvhEXLeqKTTo1eqUKKPC3eQyaKl7hLOllsB
CSDMAZOnTjC3U/dDxGkAV53ijSLdhwZAAIEJzs4bg7/fzTtxRuLWZscFs3YnFo97
nh6Vfe63SKMI2tavegw5BmV/Sl0fvBf4q77uKNd0f3p4mVmFaG5cIzJLv07A6Fpt
43C/dxC//AH2hdmoRBBYMql1GNXRor5H4idq9Joz+EkIYIvUX7Q6hL+hqkpMfT7P
T19sdl6gSzeRntwi5m3OFBqOasv+zbMUZBfHWymeMr/y7vrTC0LUq7dBMtoM1O/4
gdW7jVg/tRvoSSiicNoxBN33shbyTApOB6jtSj1etX+jkMOvJwIDAQABo2MwYTAO
BgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUA95QNVbR
TLtm8KPiGxvDl7I90VUwHwYDVR0jBBgwFoAUA95QNVbRTLtm8KPiGxvDl7I90VUw
DQYJKoZIhvcNAQEFBQADggEBAMucN6pIExIK+t1EnE9SsPTfrgT1eXkIoyQY/Esr
hMAtudXH/vTBH1jLuG2cenTnmCmrEbXjcKChzUyImZOMkXDiqw8cvpOp/2PV5Adg
06O/nVsJ8dWO41P0jmP6P6fbtGbfYmbW0W5BjfIttep3Sp+dWOIrWcBAI+0tKIJF
PnlUkiaY4IBIqDfv8NZ5YBberOgOzW6sRBc4L0na4UU+Krk2U886UAb3LujEV0ls
YSEY1QSteDwsOoBrp+uvFRTp2InBuThs4pFsiv9kuXclVzDAGySj4dzp30d8tbQk
CAUw7C29C79Fv1C5qfPrmAESrciIxpg0X40KPMbp1ZWVbd4=
-----END CERTIFICATE-----
---
Server certificate
subject=/C=US/ST=California/L=San Francisco/O=Airbnb, Inc./CN=airbnb.com
issuer=/C=US/O=DigiCert Inc/CN=DigiCert SHA2 Secure Server CA
---
No client certificate CA names sent
Server Temp Key: ECDH, P-256, 256 bits
---
SSL handshake has read 5061 bytes and written 345 bytes
---
New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    Session-ID: D2F505134A8189DB0A1C39B305819070F69E37E44809EDDC4811CA376EA83310
    Session-ID-ctx:
    Master-Key: BE70C8F66DD5FB76BF2F880D080A3F773012BA1C285E98D72C333AE38CF4F204EB58A176B62E7246B7610B466AD93BAC
    TLS session ticket lifetime hint: 600 (seconds)
    TLS session ticket:
    0000 - fb 9b 78 0b 79 d5 60 9a-6f a7 5e c4 ca b7 5f 1f   ..x.y.`.o.^..._.
    0010 - 3a ce 51 5a 0a 79 1d 54-cc a6 e6 3f eb ee 90 d0   :.QZ.y.T...?....
    0020 - cb e8 91 43 f7 74 40 0a-a0 82 cc 50 f1 50 89 46   ...C.t@....P.P.F
    0030 - 98 f1 89 f9 23 dd 6f e7-07 79 cd d2 a0 cb 14 33   ....#.o..y.....3
    0040 - 35 60 ae 1c 0b 24 0e 80-40 ab 03 41 5f a3 0b a4   5`...$..@..A_...
    0050 - 2a 1f 36 59 a0 eb 4e 18-1d ea 9c 50 c9 08 4b 44   *.6Y..N....P..KD
    0060 - 42 0f da 5a 1b 4b ec b4-43 e5 cb 44 b9 c2 24 02   B..Z.K..C..D..$.
    0070 - 25 b0 0f c8 02 63 89 48-b7 9c a8 a5 2b 55 92 cd   %....c.H....+U..
    0080 - 87 9f 02 59 26 46 b6 e4-e7 17 4d c7 6f 60 78 28   ...Y&F....M.o`x(
    0090 - cf 5a 47 aa fc db 8e bd-5c 30 76 8d 3b 48 7b 64   .ZG.....\0v.;H{d
    00a0 - 62 56 7d 35 c0 6a 52 2c-d1 a4 48 36 67 37 2d 39   bV}5.jR,..H6g7-9
    00b0 - ad 8c d6 cf 2c d6 00 23-c0 22 91 36 a5 12 84 6a   ....,..#.".6...j

    Start Time: 1604568329
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
---
read:errno=0
```

### 查看证书支持域名
可以通过浏览器查看当前域名所持有证书, 支持的域名列表。

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gkefi8tt8uj30qw0py41o.jpg)

## 其他相关知识

### HSTS
[HSTS](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) 是一套由互联网工程任务组发布的互联网安全策略机制。由于该机制的存在，证书如果配置错误可能对用户造成极大损害，因为 HSTS 缓存是需要手动清除。

#### 响应头格式
```text
Strict-Transport-Security: max-age=expireTime [; includeSubDomains] [; preload]
```

#### 浏览器处理方式
你的网站第一次通过HTTPS请求，服务器响应Strict-Transport-Security 头，浏览器记录下这些信息，然后后面尝试访问这个网站的请求都会自动把HTTP替换为HTTPS。

当HSTS头设置的过期时间到了，后面通过HTTP的访问恢复到正常模式，不会再自动跳转到HTTPS。

每次浏览器接收到Strict-Transport-Security头，它都会更新这个网站的过期时间，所以网站可以刷新这些信息，防止过期发生。

Chrome、Firefox等浏览器里，当您尝试访问该域名下的内容时，会产生一个307 Internal Redirect（内部跳转），自动跳转到HTTPS请求。

#### 浏览器操做
Chrome 浏览器内可以通过访问 chrome://net-internals/#hsts 链接，来查询域名对应 HSTS 设置。

![](https://tva1.sinaimg.cn/large/0081Kckwly1gkeg6opnjnj327n0u044i.jpg)

当然也可以手动清除浏览器本地 HSTS 缓存。

![](https://tva1.sinaimg.cn/large/0081Kckwly1gkeg8ltl3kj31ke0u0al1.jpg)

