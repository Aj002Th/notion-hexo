---
categories: 计算机原理
tags:
  - SSL
  - 网络
description: ''
permalink: ssl-man-in-the-middle-attack/
title: ssl 的中间人攻击原理
cover: /images/b8f2b57ee8b53b0920e27dc77dda26c8.jpg
date: '2023-05-29 21:50:00'
updated: '2024-04-19 21:03:00'
---

中间人攻击就是有一个中间人服务器，他一边跟客户端建立连接，冒充服务端，另外一边跟服务端建立连接，冒充客户端，实现窃听、篡改等攻击


具体的实现方式主要有两种，分别是 **ssl 嗅探** 和 **ssl 剥离**


### ssl sniff（ssl 嗅探）


`Attacker`在客户端发起连接时截获会话，用自己的公钥和公钥证书替换原本应该是由服务端提供的公钥和公钥证书。这样一来`Attacker`就可以用自己的私钥来解密客户端发送的请求，然后用来和服务端那边通信


问题：`Attacker`给客户端提供的证书毕竟不是真的证书，使用 https 单向认证，客户端去 ca 一检查就发现问题了，然后客户端可以自己选择是否继续连接，接受这个不受信任的证书。只要没有内鬼，遇到不受信任的就坚决不接受，就没有可乘之机。


### ssl Strip（ssl 剥离）


这种攻击更加复杂和麻烦，但是也更加高级和隐蔽、不需要伪造证书


`Attacker`在客户端与服务器建立连接时，在`Attacker`与服务器之间形成HTTPS连接，而在客户端与`Attacker`之间形成`HTTP`连接，即将SSL层从原HTTPS连接中“剥离”。这样，既避免了在客户端验证证书时难以避免的弹框问题，又能够劫持HTTP明文数据，并同时保证客户端`HTTP`数据的传输，达到欺骗服务器与客户端的效果。


问题：这其实依赖的是用户在访问时直接输入网址，一般直接输入的域名，而不是输入 `https://域名`，这时候默认发出去了 http 请求，就会给人这种可乘之机。服务器那边通常是在网关设置将 http 流量转换成 https 流量。客户端只要保证最开始时就是发送 https 请求，这种攻击同样是没有可乘之机。


### 情景分析

1. http 直接裸奔：直接通过 ssl sniff (ssl 嗅探) 就能直接完美实现中间人攻击，且不会被察觉
2. https 单向认证：服务端要提供证书，一般也只做到这个程度就够用了，理论上只要客户端不作死，中间人攻击就实现不了
3. https 双向认证：更高级的需求就是，服务端也要验证客户端，不是什么人都可以连接服务器，这样即使是密码什么的泄露，也无权操作服务器这边。要求客户端和服务端两边在建立连接的时候都要交换证书，去 ca 认证。通常用于企业接口对接

对于普通的 toC 业务来说 https单向认证 基本是完全可以保证安全的，部分 toB 业务要求更高的安全性，会使用 https 双向认证


### addition


[关于Https安全性问题、双向验证防止中间人攻击问题_缺少相应的安全校验很容易导致中间人攻击](https://blog.csdn.net/paolei/article/details/88028467)


[HTTPS、单向认证、双向认证，中间者攻击](https://www.jianshu.com/p/e680716ba8eb)

