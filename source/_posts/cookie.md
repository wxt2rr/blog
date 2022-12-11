---
title: 重新认识Cookie
date: 2022-12-11 17:38
categories: 
- [思考总结]
tags:
  - 重新认识系列
references:
  - title: Cookie[维基百科]
    url: https://zh.m.wikipedia.org/zh-cn/Cookie
---

本来想整理一下单点登录系统的设计，但是想了想，单点登录系统出现的意义也是为了解决不同域下共享用户登录状态（session共享）的问题。那么从源头开始，浏览器和服务器到底是如何来维护同一个用户的状态呢，老生常谈的问题，cookie和session的区别到底是什么？，由此决定从cookie开始，逐步整理关于用户登录这些事。

<!-- more -->

### Cookie

### cookie的由来

”cookie“是由网景公司的程序员卢·蒙特利创造，源自“magic cookie”（1979年就已经出现，当时指UNIX程序收发的数据包）。

卢·蒙特利在1994年6月想到这点子时正在网景工作，公司则在为[MCI](https://zh.m.wikipedia.org/wiki/世通公司)开发[电子商务](https://zh.m.wikipedia.org/wiki/电子商务)应用程序。[文顿·瑟夫](https://zh.m.wikipedia.org/wiki/文頓·瑟夫)和[约翰·克伦辛](https://zh.m.wikipedia.org/w/index.php?title=約翰·克倫辛&action=edit&redlink=1)代表MCI与网景讨论技术，表示不希望总是由其服务器保存事务状态，而要求网景将状态储存在用户的计算机中。cookie就是网景提出的解决方案。

同年，蒙特利与约翰·詹南德雷亚一起编写了最初的网景cookie规范。1994年10月13日发布的[网景导航者](https://zh.m.wikipedia.org/wiki/网景导航者)0.9beta版开始支持cookie。它公开的首次使用目的是检查网景网站的浏览者是否已经浏览过该网站。蒙特利于1995年申请了cookie技术的专利，1998年获批（[US 5774670](http://worldwide.espacenet.com/textdoc?DB=EPODOC&IDX=US5774670)）。1995年10月发布的第2版[Internet Explorer](https://zh.m.wikipedia.org/wiki/Internet_Explorer)也宣布支持cookie。

当时，cookie并未为公众所知，虽然预设用户接受cookie，网站并不会通知用户其存在。1996年2月12日，英国《[金融时报](https://zh.m.wikipedia.org/wiki/金融時報_(英國))》发表文章介绍cookie，使其为大众所知。其潜在私隐问题也引起讨论，1996年和1997年的美国[联邦贸易委员会](https://zh.m.wikipedia.org/wiki/联邦贸易委员会)两次就cookie举行听证会。

[互联网工程任务组](https://zh.m.wikipedia.org/wiki/互联网工程任务组)专门成立了一个工作小组以规范cookie的使用。布莱恩·贝伦多夫和大卫·克里斯托分别提出了两个有关HTTP事务状态的替代方案。但由克里斯托本人和蒙特利领导的小组很快决定还是使用网景规范。1996年2月，工作组将第三方cookie确定为严重私隐威胁。该小组制定的规范RFC 2109最终于1997年2月发布，要求第三方cookie要么根本不允许，要么至少预设不启用。

网景的cookie[头字段](https://zh.m.wikipedia.org/wiki/HTTP头字段)为`Set-Cookie`，[RFC 2965](https://tools.ietf.org/html/rfc2965)添加了`Set-Cookie2`头字段，即“[RFC 2965](https://tools.ietf.org/html/rfc2965) cookie”，但`Set-Cookie2`很少用，终于2011年4月的RFC 6265中弃用，已经没有现代浏览器可识别`Set-Cookie2`头字段。

所以，cookie最开始的目的是为了将用户在浏览网站时的记录信息存储在用户的设备而不是服务器上。随着cookie的流行，现在cookie的主要作用是用户识别及状态管理，web网站为了方便维护用户的状态，会通过浏览器将用户的状态信息保存到用户的终端上，然后当用户再次访问时再从用户的终端上读取用户的状态信息，从而叫web网站记住当前的用户是谁。

### cookie的组成

cookie由三部分组成：名、值、属性，如：

~~~http
HTTP/1.0 200 OK
Set-Cookie: WXTID=XXXXXX; Path=/; Expires=Wed, 13 Jan 2021 22:23:01 GMT; Secure; HttpOnly
~~~

#### 名和值

名称和值以Key=Value形式组成

#### 属性

常用属性：Domain、Path、Expires、Max-Age、Secure、HttpOnly

##### Domain和Path

`Domain`和`Path`决定了cookie的作用范围，`Domain`决定了当前的cookie属于哪个网站，为了避免不同的网站之间的cookie乱用和当前域名下cookie的安全性。`Path`指定了当前cookie可以获取到该域下的那些目录。

对于`Domain`来说，cookie只能设置到当前资源的顶级或者其子域上，但是cookie没有完全遵守同源协议，顶级域名下的cookie信息其子域是可以获取到的。所以设置cookie时禁止给.com、.org、.cn等顶级域名设置，因为这样设置了还不如不设置来的安全。

如设置名为WXTID的cookie的`Domain`为wangxt.com，那么sso.wangxt.com和www.wangxt.com都可以获取到cookie。如下示例，我们给wangxt.com域下设置名为WXTID的cookie，使用sso.wangxt.com是可以获取改cookie的。

~~~java
Cookie cookie = new Cookie("WXTID", "www.wangxt.com");
cookie.setDomain("wangxt.com");
cookie.setPath("*");
response.addCookie(cookie);
~~~

~~~http
Response Headers
Content-Length: 0
Date: Sun, 11 Dec 2022 08:32:10 GMT
Set-Cookie: WXTID=www.wangxt.com; Domain=wangxt.com; Path=*
Request Headers
...
Cookie: WXTID=www.wangxt.com
Host: sso.wangxt.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36
....
~~~

如分别设置WXTID和WXTID2的`Domain`分别是wangxt.wangxt.com和sso.wangxt.com，那么我们使用sso.wangxt.com是无法获取到WXTID的cookie的。

~~~java
Cookie cookie = new Cookie("WXTID", "www.wangxt.com");
cookie.setDomain("www.wangxt.com");
cookie.setPath("*");
response.addCookie(cookie);

cookie = new Cookie("WXTID2", "sso.wangxt.com");
cookie.setDomain("sso.wangxt.com");
cookie.setPath("*");
response.addCookie(cookie);
~~~

~~~http
Response Headers
Content-Length: 0
Date: Sun, 11 Dec 2022 08:38:33 GMT
Set-Cookie: WXTID=www.wangxt.com; Domain=www.wangxt.com; Path=*
Set-Cookie: WXTID2=sso.wangxt.com; Domain=sso.wangxt.com; Path=*
Request Headers
...
Cookie: WXTID2=sso.wangxt.com
Host: sso.wangxt.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36
...
~~~

##### Expires和Max-Age

`Expires`定义了cookie的过期时间，也就是浏览器从用户的终端删除cookie的时间。`Expires`为指定时间，格式为`Wdy, DD Mon YYYY HH:MM:SS GMT`，或者`Wdy, DD Mon YY HH:MM:SS GMT`（YY大于或等于0并小于或等于69）。使用场景是定期删除，如每天的12点。

`Max-Age`定义了cookie从浏览器接收cookie的时间开始，经过`Max-age`时长之后过期。使用场景是活期删除，如从每个用户登录之后的7天之后删除。但是`Max-age`可能存在部分浏览器（如Internet Explorer）不支持的情况。

##### Secure和HttpOnly

`Secure`决定了是否对cookie进行加密，也就是是否只能通过Https进行获取。

`HttpOnly`决定了是否只能通过Http协议（当前也包括Https）获取cookie，比如开启了`HttpOnly`，通过脚本语言则无法获取cookie信息，也避免了一些不法分子进行跨站点脚本攻击。

#### cookie的分类

##### 会话cookie

`会话cookie`也叫临时cookie，其生命周期只在浏览器和服务器的某次会话期间，并且不会保存到用户的设备中（内存memory），关闭浏览器之后就会被删除。不设置过期时间的都是`会话cookie`。

##### 持久cookie

持久cookie顾名思义，在创建的时候会设置其过期时间，并且存储到用户的设备中（硬盘risk），在过期之前都可以重复获取该cookie。

##### 安全cookie

设置了Secure为true的cookie，只能通过https进行加密传输，相对于http传输方式来说比较安全。

#### cookie的用途

~~~markdown
Cookies and similar technologies like pixels and local storage provide you with a better, faster, and safer experience on Twitter. Cookies are also used to operate our services, which include our websites, applications, APIs, pixels, embeds, and email communications. Specifically, Twitter uses these technologies to:
* Keep you logged in to Twitter.
* Deliver features and functionality of Twitter services.
* Save and honor your preferences.
* Personalize the content you see.
* Protect you against spam and abuse.
* Show you more relevant ads.
* Provide subscription features and distribute certain content.
* Understand how you interact with our services and where we can improve.
* Measure the effectiveness of our advertising and marketing.
* Rate how our services perform and identify bugs and other quality issues. 
* Gather data used to operate our business — from measuring our audience size to enforcing the Twitter Rules. 
~~~

以上是摘自Twitter网站对于cookie的使用，基本代表了现在大部分网站对于cookie的使用场景，大概为：

##### 会话管理

管理用户的会话，一般用于维护用户的登录状态，不需要每次都需要进行密码登录。当用户首次通过密码登录之后，服务器会向浏览器发送唯一标识当前会话（用户）的cookie，并且将cookie的过期时间设置为n天，下次用户再次访问该网站时，浏览器会将cookie信息带给服务器，服务器通过cookie中携带的唯一标识自动给用户进行登录等操作。

##### 喜好设定

某些网站会将用户的喜好存放在cookie中，等用户下次浏览时按照之前存储的喜好信息复原。

##### 信息收集

通过cookie跟踪用户的行为，如可以增加用户埋点上报的cookie，cookie中携带当前用户的唯一标识，在用户浏览或者点击到关键点时进行数据上报，这样可以通过cookie将用户在网站的整个操作链路记录下来，后续进行数据分析。

### 项目源码

[项目源码](https://github.com/wxt2rr/wxt-sso)