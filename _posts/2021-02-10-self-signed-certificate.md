---
layout: post
title: "Self-signed certificate"
author: "Gaius"
categories: Devops
tags: [documentation]
image: self-signed-certificate.jpg
---

## 介绍

CA 提供证书保证传输信息安全性。当然个人也可以扮演 CA，但客户端这时是不信任的, 需要将 CA 证书，即 CA 公钥集成在客户端内。

## 信息安全
信息传输过程中需要保证的安全问题有：信息的保密性、信息的安全性以及双方身份的识别。

### 信息保密性
需要两个密钥对，非对称加密密钥对 A 和对称加密密钥 B。客户端使用 A 的公钥对 B 的密钥进行加密生成 C。然后将 C 传输给服务端，服务端使用 A 的私钥对 C 进行解密，得到 B 的密钥。
则客户端和服务端都存在 B 的密钥，则可以在传输信息前，通过非对称加密密钥 B 进行信息加解密。A 的作用是保障 B 的安全传输，B 的作用是保障信息的安全传输。

### 信息完整性
需要非对称加密密钥对 B。客户端使用散列算法计算传输内容的 hash 值，即为摘要 H1。然后使用 B 的公钥对摘要进行加密, 生成加密后的摘要 C。将 C 传输给服务端，服务端用 B 的私钥对 C 进行解密, 则得到摘要 H1。
服务端再将传输过来的内容, 使用客户端相同的散列算法计算其 hash 值, 即为摘要 H2, 则比较 H1 和 H2 的值来保证信息的完整性。

### 双方身份的识别
需要非对称加密密钥对 B。服务端将 B 的公钥发送给客户端，服务端再将自己的身份内容使用 B 的私钥进行加密发送给客户端。客户端使用 B 的公钥进行解密就能认证服务端的身份。
当然这种情况可能会被第三方劫持。第三方劫持就是服务端发送给客户端信息和 B 的公钥，但中间被三方劫持到 B 的公钥。然后三方再自己创建非对称加密密钥对 C，用来和客户端交互加解密使用。则服务端发来 B 私钥加密内容，三方劫持后使用 B 公钥进行解密，然后将信息以及 C 的公钥发送给客户端。则客户端用 C 的公钥进行加密信息，三方劫持使用 C 的私钥进行解密。

## 数字证书
数字证书本身解决的就是双方身份识别/身份认证的问题, 本身是由 CA 进行签发的，CA 可以是权威厂商，也可以是个人签发。区别在于权威厂商的 CA 公钥默认集成在客户端，而个人签发需要将 CA 公钥手动集成到客户端。CA 作用只是签发服务端证书或下级 CA 证书。服务端证书才是真正信息传输加密过程中使用的。

数字证书主要分为三部分:
- 证书内容(即身份信息, 例如发行机构相关信息) C 
- 散列算法 A
- 加密过的密文 S

数字证书认证主要过程为:
1. 服务端将证书内容 C 通过散列算法 A 计算出 hash，即摘要。然后通过 CA 的私钥进行加密即生成加密过的密文 S。
2. 客户端发起请求时，服务端会将数字证书发送给客户端。客户端通过 CA 的公钥对数字证书的 S 进行解密得到摘要 H1。
3. 客户端将证书内容通过散列算法 A 计算出 hash，即摘要 H2。
4. 对比摘要 H1 和 H2 是否相等，如果相等则客户端就认证了服务端的身份。

## 自签证书生成

### Step1 自签发 CA

生成 CA 私钥

```bash
openssl genrsa -out ca.key 2048
```

创建 openssl 配置文件 `openssl.conf`, 注意 `basicConstraints` 的 CA 值为 `TRUE`，表示其为 CA 证书用来签发用的，不论几级 CA 证书 CA 值都为 `TRUE`。

```text
[ req ]
#default_bits		= 2048
#default_md		= sha256
#default_keyfile 	= privkey.pem
distinguished_name	= req_distinguished_name
attributes		= req_attributes
extensions               = v3_ca
req_extensions           = v3_ca

[ req_distinguished_name ]
countryName			= Country Name (2 letter code)
countryName_min			= 2
countryName_max			= 2
stateOrProvinceName		= State or Province Name (full name)
localityName			= Locality Name (eg, city)
0.organizationName		= Organization Name (eg, company)
organizationalUnitName		= Organizational Unit Name (eg, section)
commonName			= Common Name (eg, fully qualified host name)
commonName_max			= 64
emailAddress			= Email Address
emailAddress_max		= 64

[ req_attributes ]
challengePassword		= A challenge password
challengePassword_min		= 4
challengePassword_max		= 20

[ v3_ca ]
basicConstraints         = CA:TRUE
```

用 `openssl.conf` 配置文件在加上私钥生成自签名的 CA 证书, 即 CA 的公钥。用作客户端解密服务端的密文 S 的。

```bash
openssl req -new -key ca.key -nodes -out ca.csr -config openssl.conf
openssl x509 -req -days 36500 -extfile openssl.conf -extensions v3_ca -in ca.csr -signkey ca.key -out ca.crt
```

查看生成 CA 证书详细内容。
```bash
openssl x509 -in ca.crt -text -noout
```

### Step2 使用自签发 CA，签发服务端证书

生成服务端私钥

```bash
openssl genrsa -out sca.key 2048
```

生成服务端身份信息 CSR 文件
```bash
openssl req -new -key sca.key -out sca.csr
```

生成服务端证书。
```bash
openssl x509 -req -in sca.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out sca.crt -days 36500
```

### Step3 客户端集成 CA 公钥
这里的 CA 公钥即为 CA 的证书。
