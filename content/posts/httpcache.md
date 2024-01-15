---
title: http 缓存
date: 2018-03-15 10:32:47
tags: 
- http-cache
categories:
- HTTP
---

## 简介
重用已获取的资源能够有效的提升网站与应用的性能。Web 缓存能够减少延迟与网络阻塞，进而减少显示某个资源所用的时间。借助 HTTP 缓存，Web 站点变得更具有响应性。

## 各种类型的缓存
缓存是一种保存资源副本并在下次请求时直接使用该副本的技术。当 web 缓存发现请求的资源已经被存储，它会拦截请求，返回该资源的拷贝，而不会去源服务器重新下载。（这部分是浏览器实现的功能）这样带来的好处有：缓解服务器端压力，提升性能(获取资源的耗时更短了)。对于网站来说，缓存是达到高性能的重要组成部分。缓存需要合理配置，因为并不是所有资源都是永久不变的：重要的是对一个资源的缓存应截止到其下一次发生改变（即不能缓存过期的资源）。

缓存的种类有很多,其大致可归为两类：私有与共享缓存。共享缓存存储的响应能够被多个用户使用。私有缓存只能用于单独用户。本文将主要介绍浏览器与代理缓存，除此之外还有网关缓存、CDN、反向代理缓存和负载均衡器等部署在服务器上，为站点和 web 应用提供更好的稳定性、性能和扩展性。

## (私有)浏览器缓存
私有缓存只能用于单独用户。你可能已经见过浏览器设置中的“缓存”选项。浏览器缓存拥有用户通过 HTTP 下载的所有文档。这些缓存为浏览过的文档提供向后/向前导航，保存网页，查看源码等功能，可以避免再次向服务器发起多余的请求。它同样可以提供缓存内容的离线浏览。

## (共享)代理缓存
共享缓存可以被多个用户使用。例如，ISP 或你所在的公司可能会架设一个 web 代理来作为本地网络基础的一部分提供给用户。这样热门的资源就会被重复使用，减少网络拥堵与延迟。

## 缓存操作的目标

虽然 HTTP 缓存不是必须的，但重用缓存的资源通常是必要的。然而常见的 HTTP 缓存只能存储 GET 响应，对于其他类型的响应则无能为力。缓存的关键主要包括request method和目标URI（一般只有GET请求才会被缓存）。 普遍的缓存案例:

- 一个检索请求的成功响应: 对于 GET请求，响应状态码为：200，则表示为成功。一个包含例如HTML文档，图片，或者文件的响应.
- 不变的重定向: 响应状态码：301.
- 错误响应: 响应状态码：404 的一个页面.
- 不完全的响应: 响应状态码 206，只返回局部的信息.
- 除了 GET 请求外，如果匹配到作为一个已被定义的cache键名的响应.
针对一些特定的请求，也可以通过关键字区分多个存储的不同响应以组成缓存的内容。具体参考下文关于 Vary 的信息 .

## 缓存控制

通过网络获取内容既速度缓慢又开销巨大。较大的响应需要在客户端与服务器之间进行多次往返通信，这会延迟浏览器获得和处理内容的时间，还会增加访问者的流量费用。因此，缓存并重复利用之前获取的资源的能力成为性能优化的一个关键方面。

好在每个浏览器都自带了 HTTP 缓存实现功能。您只需要确保每个服务器响应都提供正确的 HTTP 标头指令，以指示浏览器何时可以缓存响应以及可以缓存多久。

注：如果您在应用中使用 Webview 来获取和显示网页内容，可能需要提供额外的配置标志，以确保 HTTP 缓存得到启用、其大小根据用例进行了合理设置并且缓存将持久保存。务必查看平台文档并确认您的设置！

![HTTP 请求](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/images/http-request.png?hl=zh-cn)

当服务器返回响应时，还会发出一组 HTTP 标头，用于描述响应的内容类型、长度、缓存指令、验证令牌等。例如，在上图的交互中，服务器返回一个 1024 字节的响应，指示客户端将其缓存最多 120 秒，并提供一个验证令牌（“x234dff”），可在响应过期后用来检查资源是否被修改。

## 通过 ETag 验证缓存的响应

*   服务器使用 ETag HTTP 标头传递验证令牌。
*   验证令牌可实现高效的资源更新检查：资源未发生变化时不会传送任何数据。

假定在首次获取资源 120 秒后，浏览器又对该资源发起了新的请求。首先，浏览器会检查本地缓存并找到之前的响应。遗憾的是，该响应现已过期，浏览器无法使用。此时，浏览器可以直接发出新的请求并获取新的完整响应。不过，这样做效率较低，因为如果资源未发生变化，那么下载与缓存中已有的完全相同的信息就毫无道理可言！

这正是验证令牌（在 ETag 标头中指定）旨在解决的问题。服务器生成并返回的随机令牌通常是文件内容的哈希值或某个其他指纹。客户端不需要了解指纹是如何生成的，只需在下一次请求时将其发送至服务器。如果指纹仍然相同，则表示资源未发生变化，您就可以跳过下载。

![HTTP Cache-Control 示例](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/images/http-cache-control.png?hl=zh-cn)

在上例中，客户端自动在“If-None-Match” HTTP 请求标头内提供 ETag 令牌。服务器根据当前资源核对令牌。如果它未发生变化，服务器将返回“304 Not Modified”响应，告知浏览器缓存中的响应未发生变化，可以再延用 120 秒。请注意，您不必再次下载响应，这节约了时间和带宽。

作为网络开发者，您如何利用高效的重新验证？浏览器会替我们完成所有工作：它会自动检测之前是否指定了验证令牌，它会将验证令牌追加到发出的请求上，并且它会根据从服务器接收的响应在必要时更新缓存时间戳。**我们唯一要做的就是确保服务器提供必要的 ETag 令牌。检查您的服务器文档中有无必要的配置标志。**

注：提示：HTML5 Boilerplate 项目包含所有最流行服务器的[配置文件样例](https://github.com/h5bp/server-configs)，其中为每个配置标志和设置都提供了详细的注解。在列表中找到您喜爱的服务器，查找合适的设置，然后复制/确认您的服务器配置了推荐的设置。

*   每个资源都可通过 Cache-Control HTTP 标头定义其缓存策略
*   Cache-Control 指令控制谁在什么条件下可以缓存响应以及可以缓存多久。

从性能优化的角度来说，最佳请求是无需与服务器通信的请求：您可以通过响应的本地副本消除所有网络延迟，以及避免数据传送的流量费用。为实现此目的，HTTP 规范允许服务器返回 [Cache-Control 指令](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9)，这些指令控制浏览器和其他中间缓存如何缓存各个响应以及缓存多久。

注：Cache-Control 标头是在 HTTP/1.1 规范中定义的，取代了之前用来定义响应缓存策略的标头（例如 Expires）。所有现代浏览器都支持 Cache-Control，因此，使用它就够了。

![HTTP Cache-Control 示例](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/images/http-cache-control-highlight.png?hl=zh-cn)


## Cache-control
- 每个资源都可通过 Cache-Control HTTP 标头定义其缓存策略
- Cache-Control 指令控制谁在什么条件下可以缓存响应以及可以缓存多久。

从性能优化的角度来说，最佳请求是无需与服务器通信的请求：您可以通过响应的本地副本消除所有网络延迟，以及避免数据传送的流量费用。为实现此目的，HTTP 规范允许服务器返回 Cache-Control 指令，这些指令控制浏览器和其他中间缓存如何缓存各个响应以及缓存多久。

注：Cache-Control 标头是在 HTTP/1.1 规范中定义的，取代了之前用来定义响应缓存策略的标头（例如 Expires）。所有现代浏览器都支持 Cache-Control，因此，使用它就够了。

### 禁止进行缓存
缓存中不得存储任何关于客户端请求和服务端响应的内容。每次由客户端发起的请求都会下载完整的响应内容。
```
Cache-Control: no-store
Cache-Control: no-cache, no-store, must-revalidate
```
### 强制确认缓存
如下头部定义，此方式下，每次有请求发出时，缓存会将此请求发到服务器（译者注：该请求应该会带有与本地缓存相关的验证字段），服务器端会验证请求中所描述的缓存是否过期，若未过期（注：实际就是返回304），则缓存才使用本地缓存副本。
```
Cache-Control: no-cache

```
“no-cache”和“no-store”
“no-cache”表示必须先与服务器确认返回的响应是否发生了变化，然后才能使用该响应来满足后续对同一网址的请求。因此，如果存在合适的验证令牌 (ETag)，no-cache 会发起往返通信来验证缓存的响应，但如果资源未发生变化，则可避免下载（服务端返回304）。
相比之下，“no-store”则要简单得多。它直接禁止浏览器以及所有中间缓存存储任何版本的返回响应，例如，包含个人隐私数据或银行业务数据的响应。每次用户请求该资产时，都会向服务器发送请求，并下载完整的响应。（可以理解为永不缓存）
### 私有缓存和公共缓存
"public" 指令表示该响应可以被任何中间人（译者注：比如中间代理、CDN等）缓存。若指定了"public"，则一些通常不被中间人缓存的页面（译者注：因为默认是private）（比如 带有HTTP验证信息（帐号密码）的页面 或 某些特定影响状态码的页面），将会被其缓存。

而 "private" 则表示该响应是专用于某单个用户的，中间人不能缓存此响应，该响应只能应用于浏览器私有缓存中。

```
Cache-Control: private
Cache-Control: public
```
### 缓存过期机制
“max-age”
```
Cache-Control: max-age=60
```
指令指定从请求的时间开始，允许获取的响应被重用的最长时间（单位：秒）。例如，“max-age=60”表示可在接下来的 60 秒缓存和重用响应。
针对应用中那些不会改变的文件，通常可以手动设置一定的时长以保证缓存有效，例如图片、css、js等静态资源。

### 缓存验证确认
当使用了 "must-revalidate" 指令，那就意味着缓存在考虑使用一个陈旧的资源时，必须先验证它的状态，已过期的缓存将不被使用。
```
Cache-Control: must-revalidate
```
## 定义最佳 Cache-Control 策略
![缓存决策树](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/images/http-cache-decision-tree.png?hl=zh-cn)
按照以上决策树为您的应用使用的特定资源或一组资源确定最佳缓存策略。在理想的情况下，您的目标应该是在客户端上缓存尽可能多的响应，缓存尽可能长的时间，并且为每个响应提供验证令牌，以实现高效的重新验证。

## 废弃和更新缓存的响应

- 在资源“过期”之前，将一直使用本地缓存的响应。
- 您可以通过在网址中嵌入文件内容指纹，强制客户端更新到新版本的响应。
- 为获得最佳性能，每个应用都需要定义自己的缓存层次结构。
浏览器发出的所有 HTTP 请求会首先路由到浏览器缓存，以确认是否缓存了可用于满足请求的有效响应。如果有匹配的响应，则从缓存中读取响应，这样就避免了网络延迟和传送产生的流量费用。

不过，如果您想更新或废弃缓存的响应，该怎么办？例如，假定您已告诉访问者将某个 CSS 样式表缓存长达 24 小时 (max-age=86400)，但设计人员刚刚提交了一个您希望所有用户都能使用的更新。您该如何通知拥有现在“已过时”的 CSS 缓存副本的所有访问者更新其缓存？在不更改资源网址的情况下，您做不到。

浏览器缓存响应后，缓存的版本将一直使用到过期（由 max-age 或 expires 决定），或一直使用到由于某种其他原因从缓存中删除，例如用户清除了浏览器缓存。因此，构建网页时，不同的用户可能最终使用的是文件的不同版本；刚获取了资源的用户将使用新版本的响应，而缓存了早期（但仍有效）副本的用户将使用旧版本的响应。

所以，如何才能鱼和熊掌兼得：客户端缓存和快速更新？您可以在资源内容发生变化时更改它的网址，强制用户下载新响应。通常情况下，可以通过在文件名中嵌入文件的指纹或版本号来实现 - 例如 style.x234dff.css。

![缓存层次结构](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/images/http-cache-hierarchy.png?hl=zh-cn)

因为能够定义每个资源的缓存策略，所以您可以定义“缓存层次结构”，这样不但可以控制每个响应的缓存时间，还可以控制访问者看到新版本的速度。为了进行说明，我们一起分析一下上面的示例：

*   HTML 被标记为“no-cache”，这意味着浏览器在每次请求时都始终会重新验证文档，并在内容变化时获取最新版本。此外，在 HTML 标记内，您在 CSS 和 JavaScript 资产的网址中嵌入指纹：如果这些文件的内容发生变化，网页的 HTML 也会随之改变，并会下载 HTML 响应的新副本。
*   允许浏览器和中间缓存（例如 CDN）缓存 CSS，并将 CSS 设置为 1 年后到期。请注意，您可以放心地使用 1 年的“远期过期”，因为您在文件名中嵌入了文件的指纹：CSS 更新时网址也会随之变化。
*   JavaScript 同样设置为 1 年后到期，但标记为 private，这或许是因为它包含的某些用户私人数据是 CDN 不应缓存的。
*   图像缓存时不包含版本或唯一指纹，并设置为 1 天后到期。

您可以组合使用 ETag、Cache-Control 和唯一网址来实现一举多得：较长的过期时间、控制可以缓存响应的位置以及随需更新。

## 新鲜度
理论上来讲，当一个资源被缓存存储后，该资源应该可以被永久存储在缓存中。由于缓存只有有限的空间用于存储资源副本，所以缓存会定期地将一些副本删除，这个过程叫做缓存驱逐。另一方面，当服务器上面的资源进行了更新，那么缓存中的对应资源也应该被更新，由于HTTP是C/S模式的协议，服务器更新一个资源时，不可能直接通知客户端及其缓存，所以双方必须为该资源约定一个过期时间，在该过期时间之前，该资源（缓存副本）就是新鲜的，当过了过期时间后，该资源（缓存副本）则变为陈旧的。驱逐算法用于将陈旧的资源（缓存副本）替换为新鲜的，注意，一个陈旧的资源（缓存副本）是不会直接被清除或忽略的，当客户端发起一个请求时，缓存检索到已有一个对应的陈旧资源（缓存副本），则缓存会先将此请求附加一个`[If-None-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match)`头，然后发给目标服务器，以此来检查该资源副本是否是依然还是算新鲜的，若服务器返回了 [`304`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/304 "The HTTP 304 Not Modified client redirection response code indicates that there is no need to retransmit the requested resources. It is an implicit redirection to a cached resource. This happens when the request method is safe, like a GET or a HEAD request, or when the request is conditional and uses a If-None-Match or a If-Modified-Since header.") (Not Modified)（该响应不会有带有实体信息），则表示此资源副本是新鲜的，这样一来，可以节省一些带宽。若服务器通过 If-None-Match 或 If-Modified-Since判断后发现已过期，那么会带有该资源的实体内容返回。
下面是上述缓存处理过程的一个图示：
![Show how a proxy cache acts when a doc is not cache, in the cache and fresh, in the cache and stale.](https://mdn.mozillademos.org/files/13771/HTTPStaleness.png)
对于含有特定头信息的请求，会去计算缓存寿命。比如Cache-control: max-age=N的请求头，相应的缓存的寿命就是N。通常情况下，对于不含这个属性的请求则会去查看是否包含[Expires](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Expires)属性，通过比较Expires的值和头里面[Date](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Date)属性的值来判断是否缓存还有效。如果max-age和expires属性都没有，找找头里的[Last-Modified](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Last-Modified)信息。如果有，缓存的寿命就等于头里面Date的值减去Last-Modified的值除以10（注：根据rfc2626其实也就是乘以10%）。
缓存失效时间计算公式如下：
<pre>`expirationTime = responseTime + freshnessLifetime - currentAge`</pre>
上式中，`responseTime` 表示浏览器接收到此响应的那个时间点。

### 加速资源
更多地利用缓存资源，可以提高网站的性能和相应速度。为了优化缓存，过期时间设置得尽量长是一种很好的策略。对于定期或者频繁更新的资源，这么做是比较稳妥的，但是对于那些长期不更新的资源会有点问题。这些固定的资源在一定时间内受益于这种长期保持的缓存策略，但一旦要更新就会很困难。特指网页上引入的一些js/css文件，当它们变动时需要尽快更新线上资源。
web开发者发明了一种 Steve Sounders 称作加速（译者注：revving）的技术<sup>[[1]](https://www.stevesouders.com/blog/2008/08/23/revving-filenames-dont-use-querystring/) </sup>。不频繁更新的文件会使用特定的命名方式：在URL后面（通常是文件名后面）会加上版本号。加上版本号后的资源就被视作一个完全新的独立的资源，同时拥有一年甚至更长的缓存过期时长。但是这么做也存在一个弊端，所有引用这个资源的地方都需要更新链接。web开发者们通常会采用自动化构建工具在实际工作中完成这些琐碎的工作。当低频更新的资源（js/css）变动了，只用在高频变动的资源文件（html）里做入口的改动。
这种方法还有一个好处：同时更新两个缓存资源不会造成部分缓存先更新而引起新旧文件内容不一致。对于互相有依赖关系的css和js文件，避免这种不一致性是非常重要的。
![](https://mdn.mozillademos.org/files/13779/HTTPRevved.png)
加在加速文件后面的版本号不一定是一个正式的版本号字符串，如1.1.3这样或者其他固定自增的版本数。它可以是任何防止缓存碰撞的标记例如hash或者时间戳。
## 缓存验证
用户点击刷新按钮时会开始缓存验证。如果缓存的响应头信息里含有"Cache-control: must-revalidate”的定义，在浏览的过程中也会触发缓存验证。另外，在浏览器偏好设置里设置Advanced-&gt;Cache为强制验证缓存也能达到相同的效果。
当缓存的文档过期后，需要进行缓存验证或者重新获取资源。只有在服务器返回强校验器或者弱校验器时才会进行验证。

## ETags
作为缓存的一种强校验器，[`ETag`](/zh-CN/docs/Web/HTTP/Headers/ETag "ETag HTTP响应头是资源的特定版本的标识符。这可以让缓存更高效，并节省带宽，因为如果内容没有改变，Web服务器不需要发送完整的响应。而如果内容发生了变化，使用ETag有助于防止资源的同时更新相互覆盖（“空中碰撞”）。") 响应头是一个对用户代理(User Agent, 下面简称UA)不透明（译者注：UA 无需理解，只需要按规定使用即可）的值。对于像浏览器这样的HTTP UA，不知道ETag代表什么，不能预测它的值是多少。如果资源请求的响应头里含有ETag, 客户端可以在后续的所有请求的头中带上 [`If-None-Match`]("If-None-Match 是一个条件式请求首部。对于 GETGET 和 HEAD 请求方法来说，当且仅当服务器上没有任何资源的 ETag 属性值与这个首部中列出的相匹配的时候，服务器端会才返回所请求的资源，响应码为  200  。对于其他方法来说，当且仅当最终确认没有已存在的资源的  ETag 属性值与这个首部中所列出的相匹配的时候，才会对请求进行相应的处理。") 头来验证缓存。
[`Last-Modified`]("The Last-Modified  是一个响应首部，其中包含源头服务器认定的资源做出修改的日期及时间。 它通常被用作一个验证器来判断接收到的或者存储的资源是否彼此一致。由于精确度比  ETag 要低，所以这是一个备用机制。包含有  If-Modified-Since 或 If-Unmodified-Since 首部的条件请求会使用这个字段。") 响应头可以作为一种弱校验器。说它弱是因为它是一次性的。如果响应头里含有这个信息，客户端可以在后续的一次请求中带上 [`If-Modified-Since`]("If-Modified-Since 是一个条件式请求首部，服务器只在所请求的资源在给定的日期时间之后对内容进行过修改的情况下才会将资源返回，状态码为 200  。如果请求的资源从那时起未经修改，那么返回一个不带有消息主体的  304  响应，而在 Last-Modified 首部中会带有上次修改时间。 不同于  If-Unmodified-Since, If-Modified-Since 只可以用在 GET 或 HEAD 请求中。") 来验证缓存。
当向服务端发起缓存校验的请求时，服务端会返回 [`200`]("状态码 200 OK 表明请求已经成功. 默认情况下状态码为200的响应可以被缓存。") ok表示返回正常的结果或者 [`304`]("HTTP 304 未改变说明无需再次传输请求的内容，也就是说可以使用缓存的内容。这通常是在一些安全的方法（safe），例如GET 或HEAD 或在请求中附带了头部信息： If-None-Match 或If-Modified-Since。") Not Modified(不返回body)表示浏览器可以使用本地缓存文件。304的响应头也可以同时更新缓存文档的过期时间。

## 带Vary头的响应
 Vary 是一个HTTP响应头部信息，它决定了对于未来的一个请求头，应该用一个缓存的回复(response)还是向源服务器请求一个新的回复。它被服务器用来表明在 content negotiation algorithm（内容协商算法）中选择一个资源代表的时候应该使用哪些头部信息（headers）.") HTTP 响应头决定了对于后续的请求头，如何判断是请求一个新的资源还是使用缓存的文件。

当缓存服务器收到一个请求，只有当前的请求和原始（缓存）的请求头跟缓存的响应头里的Vary都匹配，才能使用缓存的响应。
![The Vary header leads cache to use more HTTP headers as key for the cache.](https://mdn.mozillademos.org/files/13769/HTTPVary.png)

使用vary头有利于内容服务的动态多样性。例如，使用Vary: User-Agent头，缓存服务器需要通过UA判断是否使用缓存的页面。如果需要区分移动端和桌面端的展示内容，利用这种方式就能避免在不同的终端展示错误的布局。另外，它可以帮助google或者其他搜索引擎更好地发现页面的移动版本，并且告诉搜索引擎没有引入[Cloaking](https://en.wikipedia.org/wiki/Cloaking)。
`Vary: User-Agent
因为移动版和桌面的客户端的请求头中的User-Agent不同， 缓存服务器不会错误地把移动端的内容输出到桌面端到用户。
Vary的意义在于告诉代理服务器/缓存/CDN，如何判断请求是否一样，Vary中的组合就是服务器/缓存/CDN判断的依据，比如Vary中有User-Agent，那么即使相同的请求，如果用户使用IE打开了一个页面，再用Firefox打开这个页面的时候，CDN/代理会认为是不同的页面，如果Vary中没有User-Agent，那么CDN/代理会认为是相同的页面，直接给用户返回缓存的页面，而不会再去web服务器请求相应的页面。

## 参考
- https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn
- https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching_FAQ
- https://tools.ietf.org/html/rfc7234
- https://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13
- https://www.mnot.net/cache_docs
- http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html