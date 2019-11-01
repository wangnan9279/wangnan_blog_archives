---
title: 为什么要用 HTTPS
link_title: net-why-https
date: 2017-09-19 14:05:04
tags: [网络]
categories: 网络
thumbnailImage: https://i.loli.net/2019/09/24/BWbJOq9DaQA7gcE.png	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://i.loli.net/2019/09/24/BWbJOq9DaQA7gcE.png)
<!-- toc -->
# 前言
HTTP 是一种超文本传输协议，它是无状态的、简单快速的、基于 TCP 的可靠传输协议。

缺点： HTTP 是明文传输的，这就造成了很大的安全隐患

让自己变得更安全，从源头来控制风险。这就诞生了 HTTPS 协议

# HTTP 三大风险：
1. 窃听风险（eavesdropping）：第三方可以获知通信内容。
2. 篡改风险（tampering）：第三方可以修改通信内容。
3. 冒充风险（pretending）：第三方可以冒充他人身份参与通信。

# HTTPS 解决方案
1. 所有信息都是加密传播，第三方无法窃听。
2. 具有校验机制，一旦被篡改，通信双方会立刻发现。
3. 配备身份证书，防止身份被冒充。



# HTTPS
HTTP 不就是因为明文传输，所以造成了安全隐患。那让数据传输以加密的方式进行，不就消除了该隐患

从网络的七层模型来看，原先的四层 TCP 到七层 HTTP 之间是明文传输，在这之间加一个负责数据加解密的传输层（SSL/TLS），保证在网络传输的过程中数据是加密的，而 HTTP 接收到的依然是明文数据，不会有任何影响。

```
http<->SSL/TLS<->TCP
```

SSL是Netscape开发的专门用户保护Web通讯的，目前版本为3.0。最新版本的TLS 1.0是IETF(工程任务组)制定的一种新的协议，它建立在SSL 3.0协议规范之上，是SSL 3.0的后续版本。两者差别极小，可以理解为SSL 3.1，它是写入了RFC的。

# HTTPS 是如何做到数据的加密传输
## 加密算法
常用的有两种：对称加密与非对称加密
- 对称加密：即通信双方通过相同的密钥进行信息的加解密。加解密速度快，但是安全性较差，如果其中一方泄露了密钥，那加密过程就会被人破解。
- 非对称加密：相比对称加密，它一般有公钥和私钥。公钥负责信息加密，私钥负责信息解密。两把密钥分别由发送双发各自保管，加解密过程需两把密钥共同完成。安全性更高，但同时计算量也比对称加密要大很多。

## 过程
见原文图，
几个概念： 
- sessionkey  
- 公钥私钥 
- 数字证书


# 证书
## 按验证级别分类
- Domain Validation（DV）：最基本的验证方式，只保证访问的域名是安全的。但在证书中不会提及任何公司/组织信息，所以这算是最低级别的验证。
- Organization Validation（OV）：这一级别的证书解决了上面域名与公司无法对应的问题，该证书中会描述公司/组织的相关信息。确保域名安全的同时，也保证了域名与公司的绑定关系。
- Extended Validation（EV）：该级别的证书具有最高的安全性，也是最值得信任的。它除了给出更详细的公司/组织信息外，还在浏览器的地址栏上直接给出了公司名称。
- 
## 按覆盖级别分类
- 单域名 SSL 证书：这类证书只能针对一个域名使用，如，当证书与 yourdomain.com 配对了，那就不能在用在 sub.yourdomain.com 上了。
- 通配符域名 SSL 证书：这类证书可以覆盖某个域名下的所有子域名。如，当证书与 yourdomain.com 配对了，那默认 sub.yourdomain.com 以及其他子域名都加入了安全验证。
- 多多域名 SSL 证书：这类证书可以使用在多个不同的域名上。如，域名 a.com 与 b.com 上可以配对同一个证书。

# 从Http迁移的注意点
- 首先需要从相关的网站申请需要的证书；
- 在服务器上开放 HTTPS 所需的443端口，并设置相应的证书地址，下面贴一段在 nginx 上的配置：
```xml
server {
    listen 443;
    server_name localhost;
    ssl on;
    root html;
    index index.html index.htm;
    ssl_certificate   cert/证书文件.pem;
    ssl_certificate_key  cert/证书私钥.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        root html;
        index index.html index.htm;
    }
}
```
- 对于原先的80端口，需做重定向的跳转。将原先访问 HTTP 协议的用户，自动跳转用 HTTPS 上来。
- 出于性能考虑，Session Key 的生成相对耗 CPU 的资源，所以尽量减少 Key 的生成次数。这里有两种方案：1.采用长连接方式 2.在会话创建时，对生成的 Session Key 进行缓存

# 原文
https://github.com/jasonGeng88/blog/blob/master/201705/https.md

