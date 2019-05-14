### 关于在GET请求中使用body

故事还得从一个bug说起。今天有人问我，为什么发到后端的请求400了，我说肯定是参数不对，你去检查检查GET、POST之类的方法写没写对，要么就是字段没对上，无非是这几个问题。然后他说检查过了，没问题啊；我不太相信，但是看了看前端发送的请求，好像确实没啥问题：
<img src="https://img-blog.csdnimg.cn/20190506193419183.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlcm1pdFN1bg==,size_16,color_FFFFFF,t_70" width="75%" alt=""/>
我说既然这样，那肯定是后端写错了，但后端说他已经用postman测过了，肯定没问题。这就很有意思了。于是我要来了后端的代码：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190506193634686.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlcm1pdFN1bg==,size_16,color_FFFFFF,t_70)不出所料，后端把GET请求里的参数当成body的内容了，把`@RequestBody`改成`@RequestParam`应该就没问题了；改完之后果然好了。

这本来是一个很简单的问题，没什么可说的，但是他们接着问我，为什么GET请求里不能用body？我寻思着平时也没什么人在GET请求里加body吧，而且一直以来都听说这么用不好。但是为什么不好呢？所以我就查了一些资料，最后干脆又去RFC翻了翻。看到官方是这么说的：

> [RFC7231] A payload within a GET request message has no defined semantics; sending a payload body on a GET request might cause some existing implementations to reject the request.

意思是你往GET请求里加body是不符合规范的，不保证所有的实现都支持（主要是以前的实现，因为以前曾经有相应的规定），要是出了啥问题别怪我没提醒你。而且据说老版本的postman是不支持在GET请求里加body的，也是最近才加上的支持；所以要放在以前也就没这些问题了，以前的postman根本发不了带body的GET请求。

但是这一条并不是强制规定。我看很多人都强调一点，GET请求不应携带请求体，服务器应忽略（或者说丢弃）GET请求的请求体。这一条的确是有依据的，来源如下：

> [RFC2616] A server SHOULD read and forward a message-body on any request; if the request method does not include defined semantics for an entity-body, then the message-body SHOULD be ignored when handling the request.

当然，官方也只是说SHOULD，没有像前文一样措辞严厉地强调类似HEAD这种的请求MUST NOT have a message body:

> [RFC2616]  A message-body MUST NOT be included in a request if the specification of the request method does not allow sending an entity-body in requests. 

但是很可惜的是，RFC2616已经过时了。现在的说法变成了这样，连SHOULD都直接去掉了，要求更加宽松：

> [RFC7230] Request message framing is independent of method semantics, even if the method does not define any use for a message body.

难道是因为错的人多了，错的也变成了对的？不过即使是这样，也并不是在GET请求里加上body的理由。

这个问题算是解决了。但是我看到网上各路大神说到GET加body的时候，还提到一个，就是往GET里加body会导致缓存机制失效。“GET 被设计来用 URI 来识别资源，如果让它的请求体中携带数据，那么通常的缓存服务便失效了，URI 不能作为缓存的 Key。”我一开始以为是在服务器上配置的缓存，比如Nginx的cache之类的机制，后来发现好像并非如此。

那么，GET和POST的区别和应用？这问题挺复杂。简而言之，就是“安全”和“不安全”的区别。什么是安全？不用承担责任。什么是不安全？可能需要承担责任。举个例子，点击某个链接以同意某个协议，这个请求明显就是不安全的，因为需要承担责任。如果采用GET，就违反了GET应该用于安全请求的规范。因为浏览器可能在你不知情的情况下预加载这个页面（因为是“安全”的GET请求），这样相当于你在不知情的情况下同意了某个协议，这显然是我们不希望看到的。在契约式的设计里，违反契约的行为是会带来严重的后果的。浏览器按照契约预加载了安全的GET请求，但这本身是不安全的，带来的后果自然要由打破契约的人承担（将这个请求设计成GET的人出来挨打）。

之所以强调“安全”，而不是按照常规的说法强调副作用，因为有副作用的请求不代表不安全；举例来说，服务器有一个显示访问人数的功能，这个功能就可以用GET来做。虽然每次访问都会发送改变服务器状态（计数器）的请求，但用户不会因为这个请求承担责任，这个请求是安全的。至于什么GET请求的URL有长度限制（后来事实证明其实没有），什么GET请求的URL里不能有中文（或者说非ASCII吧），都只是实现上的区别；从最初的设计上来说区别并不在这里。

当然，这些都是纯粹的理论层面的东西。如果遵守RESTful的规范，采用语义化的GET/POST请求，自然也就不会有这些问题了。因为通常来说，查询是安全的；这也是GET的主要作用。

说起来也挺有意思的，学习了这么久，经常提起RFC，也没搞清楚RFC究竟是个啥玩意，这次就一并查了。虽然我总觉得这是受到6f名词解释的影响……原来是叫“Request For Comments”。

------

参考文章（不分先后）：

1. [不再以讹传讹，GET和POST的真正区别](http://www.nowamagic.net/librarys/veda/detail/1919)
2. [HTTP GET with request body](https://stackoverflow.com/questions/978061/http-get-with-request-bod)
3. [谁说 HTTP GET 就不能通过 Body 来发送数据呢？](https://yanbin.blog/why-http-get-cannot-sent-data-with-reuqest-body)
4. [URIs, Addressability, and the use of HTTP GET and POST](https://www.w3.org/2001/tag/doc/whenToUseGet.html)

此外，'URIs, Addressability, and the use of HTTP GET and POST'这篇文章我[翻译成了中文](https://blog.csdn.net/HermitSun/article/details/89880161)，欢迎各位阅读并指正；翻译水平实在有限，只能说“尽最大努力交付”。