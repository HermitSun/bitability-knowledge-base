### 关于JavaScript的重载实现

在写数据库操作的时候，习惯性地想按照Java的写法写几个重载的函数，突然发现JavaScript好像没有重载这个操作……但是不用重载感觉整个人都不舒服，所以就去查了一下，还真的看到了一点有趣的东西，就记录一下。

其实最容易想到的是利用`arguments`和`switch`来实现重载的效果，但是这种写法太丑陋了，所以不予考虑：

```javascript
function foo() {
　　switch(arguments.length) {
　　　　case 0:
          //doSomething
　　　　　　break;
　　　　case 1:
          //doSomething
　　　　　　break;
　　　　case 2:
          //doSomething
          break;
       //more cases...
}
```

然后我在网上看到了一种很精妙的写法（来源据说是《Secrets of the JavaScript Ninja》，从[这篇博客](https://www.cnblogs.com/yugege/p/5539020.html)搬来的），摘录如下：

```javascript
function addMethod(object, name, fn) {
　　var old = object[name];
　　object[name] = function() {
　　　　if(fn.length === arguments.length) {
　　　　　　return fn.apply(this, arguments);
　　　　} else if(typeof old === "function") {
　　　　　　return old.apply(this, arguments);
　　　　}
　　}
}
 
var people = {
　　values: ["Dean Edwards", "Alex Russell", "Dean Tom"]
};
 
// 不传参数时，返回people.values里面的所有元素
addMethod(people, "find", function() {
　　return this.values;
});
 
// 传一个参数时，按first-name匹配的进行返回
addMethod(people, "find", function(firstName) {
　　var ret = [];
　　for(var i = 0; i < this.values.length; i++) {
　　　　if(this.values[i].indexOf(firstName) === 0) {
　　　　　　ret.push(this.values[i]);
　　　　}
　　}
　　return ret;
});
 
// 传两个参数时，返回first-name和last-name都匹配的元素
addMethod(people, "find", function(firstName, lastName) {
　　var ret = [];
　　for(var i = 0; i < this.values.length; i++) {
　　　　if(this.values[i] === (firstName + " " + lastName)) {
　　　　　　ret.push(this.values[i]);
　　　　}
　　}
　　return ret;
});

console.log(people.find()); //["Dean Edwards", "Alex Russell", "Dean Tom"]
console.log(people.find("Dean")); //["Dean Edwards", "Dean Tom"]
console.log(people.find("Dean Edwards")); //["Dean Edwards"]
```

这种写法充分利用了闭包的特性，用类似递归的方式实现了重载（因为每次参数不匹配都会调用上一层的`old`）。[这篇文章](https://www.cnblogs.com/yugege/p/5539020.html)对这个函数的原理解释得很清楚，我就不再多说了。

我要说的是，在评论区看到了一个很有意思的问题：如果换成箭头函数，为什么不行？评论区也没有给出一个合理的解释。其实原理很简单，就是`this`的指向问题。我把上面的代码改了改，在Node和浏览器环境中分别测试了一下，结果如下（运行环境是Node，v10.15.0）：

```javascript
addMethod(people, "find", () => {
    return this.values;
});

addMethod(people, "find", (firstName) => {
    var ret = [];
    for (var i = 0; i < this.values.length; i++) {
        if (this.values[i].indexOf(firstName) === 0) {
            ret.push(this.values[i]);
        }
    }
    return ret;
});

addMethod(people, "find", (firstName, lastName) => {
    var ret = [];
    for (var i = 0; i < this.values.length; i++) {
        if (this.values[i] === (firstName + " " + lastName)) {
            ret.push(this.values[i]);
        }
    }
    return ret;
});

console.log(this.values);
console.log(people.find());
console.log(people.find("Dean"));
console.log(people.find("Dean Edwards"));
//Node: undefined
//      TypeError: Cannot read property 'length' of undefined
//FireFox: undefined
//         TypeError: this.values is undefined
//Chrome: ƒ values(object) { [Command Line API] }
//        []
//        []
```

从输出中可以看出问题所在，就是`this.values`未定义。但是为什么会未定义？原因很简单，箭头函数修改了`this`的机制，强制让箭头函数内的`this`指向外层作用域，也就是说，这里指向的是全局（`window`，而非`global`，我在上一篇文章里提到过这一点），所以输出`undefined`并且`this.values`就顺理成章了，因为在全局里确实没有定义。

但在Chrome环境中运行的时候的输出非常有趣，会输出`ƒ values(object) { [Command Line API] }`。这是Chrome的一个“黑科技”，官方文档如下：

>The Command Line API contains a collection of convenience functions for performing common tasks: selecting and inspecting DOM elements, displaying data in readable format, stopping and starting the profiler, and monitoring DOM events.
>
>**Note:** This API is only available from within the console itself. You cannot access the Command Line API from scripts on the page.
>
>……
>
>###### values(object)
>
>`values(object)` returns an array containing the values of all properties belonging to the specified object.

这是Chrome内置的一套函数，而且只会在控制台出现；这里的出现是碰巧命名重复了。它的出现也印证了前文的观点，即此处的`this`指向的是全局（不然`this.values`就不会返回`Command Line API`）。至于后面为什么没有报错，而是返回了`[]`，原因很简单，`this.values.length = 0`，所以直接把函数里定义的空数组返回了。这一点可以自行试验一下。

总之，`this`的问题一定要搞清楚，不然遗患无穷（笑）。