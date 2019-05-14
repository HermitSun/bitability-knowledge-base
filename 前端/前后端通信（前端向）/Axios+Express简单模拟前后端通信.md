### Axios+Express简单模拟前后端通信

今天实现了一个登录功能，为了实现登录功能，得先打通前后端通信。前端已经写好了，而后端的接口迟迟不来，心急之下，就想着自己模拟一个后端服务器试试，算是体验了一把全栈的快感。感叹一句，当初发明Node.js的人真是天才，没有他就没有什么JS全栈之说了。

关于技术选取，前端向后端的通信上，因为vue-resource已经停止了维护，再加上我看普遍认为Axios比较好用，所以采用了Axios；而后端向前端的通信上就有很多讲究了，我先后试验了JSON Server、Mock.js和Express，最后选择了Express。因为前后端传输JSON格式比较方便，所以如果是那种只传输数据，不涉及什么判断逻辑的简单通信，可以选择JSON Server，语法简单，配置也舒服。不过通常不会这么简单，所以还是要选择重量级一点的工具。其实Mock.js也很好用，但是因为这次项目用了TypeScript，导入不了Mock（很奇怪的一个bug），再加上我不喜欢Mock的语法，所以最后选择了Express。

先贴上代码（以post为例）：

```javascript
//server.js
const express = require('express');//导入express模块
const bodyParser = require('body-parser');//解析前端传过来的数据的中间件
const app = express();//初始化express

const jsonParser = bodyParser.json();//JSON解析器，根据具体传输的数据格式选择，比如还有：
// const urlencodedParser = bodyParser.urlencoded({extended: false});

app.use(function (req, res, next) {
    res.header("Access-Control-Allow-Origin", "*");
    res.header('Access-Control-Allow-Methods', 'PUT, GET, POST, DELETE, OPTIONS');
    res.header("Access-Control-Allow-Headers", "X-Requested-With");
    res.header('Access-Control-Allow-Headers', 'Content-Type');
    next();
});//解决通信跨域问题，直接复制就行;use是express注册中间件的方法

app.post('/LoginController/login', jsonParser, (req, res) => {
    //doSomething
    res.json(...);//返回JSON
});

app.listen(3000, () => {
    console.log('I\'m listening on port 3000!')
});//监听3000端口
```

```javascript
//axois配置
axios.post('http://localhost:3000/LoginController/login', {
          //JSON数据
        }).then((response) => {
          console.log(response.data)//返回的数据在这里
    	  //doSomething
        }).catch((error) => {
          console.log(error)//捕获错误
        })
```

在开始之前先插一句，如果`app.get()`提示`Unresolved function or method get()`，在终端安装`npm install --save-dev @types/express`即可解决（强迫症患者福利）。

为什么要有解析器？因为据说Express 4.x版本把解析器从主程序里移除了，所以需要单独添加依赖；如果没有解析器，会出现收到请求但请求内容为空的bug。

Axios的使用基本上就是这个形式，不过是操作不同罢了。官方文档是这么说的：

>为方便起见，为所有支持的请求方法提供了别名：
>
>axios.request(config)
>
>axios.get(url[, config])
>
>axios.delete(url[, config])
>
>axios.head(url[, config])
>
>axios.post(url[, data[, config]])
>
>axios.put(url[, data[, config]])
>
>axios.patch(url[, data[, config]])

我觉得这种别名的用法比较好，因为意图很明显，比在内部声明`method`好。另外，使用get的时候最好采用

`axios.get('url',{params: {}})`，这样可以确保参数出现在URL里。这是用Axios发送请求。

而用Axios接收信息，也就是`.then()`的内容，是这样的：

```javascript
{
  // 服务器响应的数据
  data: {},
  // 服务器响应的 HTTP 状态码
  status: 200,
  // 服务器响应的 HTTP 状态信息
  statusText: 'OK',
  //服务器响应头
  headers: {},
  //为请求提供的配置信息
  config: {}
}
```

基本上就是获取`response.data`的内容并进行操作了。

Express与Axios使用方法上的区别，最明显的可能就是Axios要在两个函数里分别处理`req`和`res`，而Express则是在同一个函数里处理。关于Express的基本请求，也就是参数`req`的处理，我觉得[这篇文章](https://www.jianshu.com/p/f219ff84c5e5)写得很好。文章内容如下：

>路径请求及对应的获取路径有以下几种形式：

```javascript
#req.query

// GET /search?q=tobi+ferret  
req.query.q  
// => "tobi ferret"  

// GET /shoes?order=desc&shoe[color]=blue&shoe[type]=converse  
req.query.order  
// => "desc"  

req.query.shoe.color  
// => "blue"  

req.query.shoe.type  
// => "converse"  

#req.body

// POST user[name]=tobi&user[email]=tobi@learnboost.com  
req.body.user.name  
// => "tobi"  

req.body.user.email  
// => "tobi@learnboost.com"  

// POST { "name": "tobi" }  
req.body.name  
// => "tobi"  

#req.params

// GET /user/tj  
req.params.name  
// => "tj"  

// GET /file/javascripts/jquery.js  
req.params[0]  
// => "javascripts/jquery.js" 

#req.param(name)

// ?name=tobi  
req.param('name')  
// => "tobi"  

// POST name=tobi  
req.param('name')  
// => "tobi"  

// /user/tobi for /user/:name   
req.param('name')  
// => "tobi"  
```

上面提到了针对不同方式的`req`处理方式。至于`res`，是根据不同类型的数据调用返回方法，一般是选择JSON，所以这里用的是`res.json()`。所有的方法如下：

```javascript
res.download()	//提示下载文件。
res.end()		//终结响应处理流程。
res.json()		//发送一个 JSON 格式的响应。
res.jsonp()		//发送一个支持 JSONP 的 JSON 格式的响应。
res.redirect()	//重定向请求。
res.render()	//渲染视图模板。
res.send()		//发送各种类型的响应。
res.sendFile	//以八位字节流的形式发送文件。
res.sendStatus()//设置响应状态代码，并将其以字符串形式作为响应体的一部分发送。
```

更多的一些东西可以参考[这篇文章](https://www.cnblogs.com/mq0036/p/5243312.html)，可以说写得相当全面了。

