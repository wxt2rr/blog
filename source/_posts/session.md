---
title: 重新认识Session
date: 2022-12-11 22:16
categories: 
- [思考总结]
tags:
  - 重新认识系列
references:
  - title: 重新认识cookie[blog.wangxt.online]
    url: https://blog.wangxt.online/2022/12/11/cookie/
---



上篇提到了cookie的由来及使用场景，因为HTTP是一种无状态协议，服务器不会存储任何关于用户的信息。为了给用户更好的使用体验，我们可以将这些信息通过cookie保存在用户的终端上，以便于下次用户再次打开网站时以cookie方式带给服务器。cookie已经实现了我们想要的功能，那么为什么还会有session呢，session到底是什么呢，本篇将谈一谈何为session。

<!-- more -->

### session的由来

和cookie一样，同样是为了解决HTTP无状态的问题，相对于cookie来说，session显得更加抽象。session直译过来就是会话，我们尝试把客户端和服务器的一次完整的数据交互称为一次会话，那么会话的生命周期就是在浏览器首次访问服务器之后，浏览器关闭之前，所以相对其cookie来说，session有着更加明确的生命周期界限。所以，sessionId的出现解决了浏览器和服务器一次会话周期内的用户状态的维护。我们用session去解决多次会话之间的用户状态维护，其实是无法直接使用web应用服务器原生的session机制做到的。

除了上边说到的，还有个现实的问题就是有些浏览器不支持cookie，那么你cookie在好用也白搭，怎么达到叫服务器认识用户的效果呢？当用户首次通过浏览器访问服务器时，服务器会将用户的信息保存到服务器上（不同服务器方式不同，如tomcat是存在内存中，php是存在文件里），通常存储方式是sessionId=userinfo的键值对，然后服务器会将sessionId返回给浏览器，当用户访问其它页面时，浏览器在将sessionId携带过去，这样服务器通过sessionId便知道你是谁了。

### sessionId的实现方式

sessionId是由web应用服务器创建的，不同的web应用服务器创建的规则不同，目的就是保证唯一，因为sessionId的创建和客户端没有任何关系，所以只要保证sessionId生成的唯一就可以，只要客户端请求时没有带给服务器sessionId，那么服务器就去生成一个新的就好了。常见的web应用服务器如tomcat是使用随机字符串加密（因为sessionId的生成是线程安全的，所以生成sessionId存在一定性能影响）。

不过值得注意的一点是，服务器中的session数据并不会随着浏览器的关闭而删除，只有当服务器重启或者显示的调用删除session数据的方法或者根据session服务的过期算法，session信息才会被删除。这点不要和cookie混淆，即使我们使用cookie的方式将sessionId存储，cookie过期之后，服务器中的session信息不会立马过期，只不过是客户端再去请求服务器时没有cookie，服务器又会生成一个新的session信息，并且返回一个新的sessionId到cookie里。

### sessionId传递的实现方式

上边提到服务器生成session信息后会返回给浏览器，浏览器之后的每次请求再将sessionId携带给服务器，服务器便知道是哪个用户，那么问题来了，浏览器怎么将sessionId带给服务器呢？一般使用下面几种方式：

#### cookie方式

cookie本身就可以将数据存储在用户终端里，并且每次请求将数据带给服务器，所以用cookie天然可以支持，目前大部分也是使用cookie方式。用户首次请求服务器，服务器会给用户生成唯一的sessionId通过cookie的方式返回给浏览器，浏览器根据服务器设置的过期时间对sessionId进行存储，然后后续请求在通过cookie将sessionId带给服务器，服务器根据sessionId对用户进行鉴权。

**好处：**实现方便，流程通俗易懂

**坏处：**对于不支持cookie的浏览器，面临和cookie一样的问题

#### Url重写

通过`url重写`可以把sessionId附加在HTML页面中所有的URL上，这些页面作为响应发送给浏览器。这样，当用户单击URL时，会话ID被自动作为请求行的一部分而不是作为头行发送回服务器，服务器拿到sessionId参数便可以识别用户。

**好处：**摆脱浏览器对cookie限制，即使浏览器不支持cookie或者用户主动禁止cookie，仍然不会有影响。适用于Get方式。

**坏处：**显而易见，网站所有的url后边都会拼接sessionId=xxxx，需要对其进行编码，否则部分插件可能解析有问题；html页面是静态的，所以无法对html静态页面进行重写，所有的html都需要通过servlet运行才能在发送给浏览器的时候进行url重写。用户主动删除拼接的sessionId后，无法正确得到客户端的状态。并且想要实现类似7天免登录功能，需要浏览器使用其它方式存储sessionId。

#### 隐藏表单

这种方式比较简单，在所有的页面使用隐藏的`＜input type="hidden" name="jsessionid" value="xxxxxx"＞ `，把服务器传递过来的sessionId设置到页面的隐藏表单中，对用户透明，每次请求服务器时从表单中取出sessionId的值，然后携带给服务器，服务器拿到sessionId后可以识别用户。

**好处：**和`Url重写`一样，该方式不受限于cookie，并且对用户透明。

**坏处：**需要补充设置表单的额外逻辑，并且点击`<a herf="xxx">`标签中的超链接时不会通过表单提交事件，无法做到通用的会话跟踪。和`url重写`一样，想要实现类似7天免登录功能，需要浏览器使用其它方式存储sessionId。

### cookie和session的区别

通过[重新认识cookie](https://blog.wangxt.online/2022/12/11/cookie/)和[重新认识session](https://blog.wangxt.online/2022/12/11/session/)，我们做个总结：

**cookie**

由于 HTTP 是一种无状态协议，因此它不会在其服务器上保存有关用户的任何信息。Cookie 是实现此目标的有用工具。它使我们能够将信息保存在用户的计算机上并监控正在使用的任何应用程序的状态。cookie将用户的数据存储在用户的终端上，在指定的生命周期之后自动删除。cookie是基于HTTP和浏览器的。

**session**

和cookie一样也是为了解决HTTP无状态的问题。相对于cookie来说，session信息存储在服务器，更加安全可靠。在不支持cookie的情况下，session方式是替代cookie的一个方案。另外cookie的存储是有限制的，只能存储4kb的数据，但是对于session方式来说取决于服务器的存储容量。session是基于服务器的。