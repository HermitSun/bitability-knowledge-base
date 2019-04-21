### **关于JavaScript中*this*的一些理解**

2019.2.6 修改了错误

------

`this`的使用方法已经被历代JavaScript使用者们写烂了，在此不再赘述`this`的基本使用方法，诸如默认绑定、隐式绑定、显式绑定、new绑定之类的就不说了。

我想说的是一件很有意思的事情：Node.js环境和浏览器环境下的全局变量是不一样的，Node的全局变量是`global`，而浏览器的全局变量是`window`。插一句，Node这东西实质上是对Chrome V8引擎进行了封装。这大概能解释在本地Node环境里运行时的一些神秘的bug。如下（运行环境是Node，v10.15.0）：

```javascript
function foo() {
    console.log(this.a);
}
var obj = {
    a: 2,
    foo: foo
}
var bar = obj.foo;
var a = 3;
bar();//Node环境下是undefined，但浏览器环境下是3
console.log(global.a);//还是undefined
console.log(window.a);//ReferenceError: window is not defined
console.log(this.a);//还是undefined？
```

首先需要说的是，因为Node里的全局变量是`global`，所以`window`在Node里没有定义，导致出现了`ReferenceError`。将代码进行修改：

```javascript
function foo() {
    console.log(this.a);
}
var obj = {
    a: 2,
    foo: foo
}
var bar = obj.foo;
a = 3;
bar();//3
console.log(global.a);//3
console.log(this.a);//undefined
```

简单来说，`bar`相当于是一个二级的函数指针，指向了全局的`foo()`，而`obj.foo()`直接调用则相当于来自对象的调用。从理论上来说，这个现象叫“Implicitly Lost”，即隐式绑定丢失。所谓的隐式绑定丢失，就是应用隐式绑定的函数的`this`会“莫名其妙”地指向全局对象（或者`undefined`，如果是严格模式的话）：

> Even though `bar` appears to be a reference to `obj.foo`, in fact, it's really just another reference to `foo` itself. Moreover, the call-site is what matters, and the call-site is `bar()`, which is a plain, un-decorated call and thus the *default binding* applies.
>
> 虽然`bar`是`obj.foo`的一个引用，但是实际上，它引用的是foo函数本身，因此此时的bar()其实是一个不带任何修饰的函数调用，因此应用了默认绑定。

所以，`this`指向了全局。因而在浏览器环境下输出的是`window.a`，也就是3；但在Node环境下输出结果还是`undefined`，我一度以为是`window`在Node里没有定义导致的，但事情并没有这么简单，这事后面再说。就所谓的“隐式绑定丢失”而言，我感觉这里没有丢失不丢失的说法，就是一个类似二级指针的实现而已。

`this`的指向实在是让人迷惑，所以我就去查了一下，然后就看到了下面的代码（来源是某个博客）。这段代码一度让我怀疑自己对JavaScript的理解：

```javascript
x = 100;

console.log(this.x);//undefined
this.x = 23;
console.log(this === global);//false
console.log(x)；//100

function test() {
    console.log(x);//100
    console.log(this.x);//100
    this.x = 22;
    console.log(this === global);//true
}

test();
console.log(this);//{ x: 23 }
console.log(global.x);//22
```

好在现在算是弄明白了。下面就结合起来进行一些分析，顺便也回答了之前的问题。

我们知道，在JavaScript里声明变量的时候，用`var`关键字是在当前作用域里声明一个变量，如果在函数里声明，就是局部变量，在全局环境下声明，就是全局变量；而不加`var`前缀就会直接声明一个全局变量（前提是不存在重复声明，因为这实际上是一个赋值操作，参见上次的LHS和RHS赋值原理）。与浏览器环境下的全局变量`window`不同的是，在Node中，`this`的指向分为两种情况：全局环境中，`this`指向全局对象`global`；模块环境中，`this`指向`module.exports`。

在上面的程序里声明a的时候加不加`var`看起来是没什么区别的（因为看起来都是声明一个全局变量而已），但这里涉及到一个问题，就是Node的*模块化*。

> 模块是Node应用程序的基本组成部分，文件和模块是一一对应的。换言之，一个 Node文件就是一个模块，这个文件可能是JavaScript 代码、JSON 或者编译过的C/C++ 扩展。

下面是Node官方文档对模块的说明：

> Before a module's code is executed, Node.js will wrap it with a function wrapper that looks like the following:

```javascript
(function(exports, require, module, __filename, __dirname) {
// Module code actually lives in here
});
```

> By doing this, Node.js achieves a few things: It keeps top-level variables (defined with var, const or let) scoped to the module rather than the global object.

也就是说，我们写的代码会被Node自动包装在一个函数里，导致作用域发生了变化；这就是`this`变化的罪魁祸首。实际上，Node会把.js文件当成模块，我们所写`var`声明的a的作用域只在这个模块里，上面程序里的`this`指的是`module.exports`，所以在前面两段代码里，之所以`this.a = undefined`，是因为我们根本就没有把它暴露出来，没有在`module.exports`里定义。此时`module.exports`是空的（`{}`）。可以在前两段程序里加上`module.exports.a = 4;`，然后就能看到输出变成了4。

至于第三段程序，是跟下面这一段程序等价的：

```javascript
(function (exports, require, module, __filename, __dirname) {
    x = 100;

    console.log(this.x);//undefined
    this.x = 23;
    console.log(this === global);//false
    console.log(x)；//100

    function test() {
        console.log(x);//100
        console.log(this.x);//100
        this.x = 22;
        console.log(this === global);//true
    }

    test();
    console.log(this);//{ x: 23 }
    console.log(global.x);//22
});
```

在这里，`test()`是直接调用，里面的`this`应用了默认绑定，指向了全局对象。而在外层的`this`，因为Node的机制，指向了`module.exports`。一开始`module.exports`是空的，所以输出`undefined`，而在`this.x = 23`这句语句之后，里面多了一个值为23的x，这个也可以在后面的输出里看到。至于`console.log(x)`，当然是根据RHS查询“最近”的x了。而“最近”的x就是`global.x`（`module.exports`并不影响作用域本身），所以输出的是100（很不幸这里面所有的`console.log(x)`输出的都是全局的那个x）。

想起来箭头函数了，箭头函数和`this`关键字的实现机制是完全不同的。this是基于调用栈的，而箭头函数则是基于作用域的，固定指向外层作用域。举个例子：

```javascript
function foo() {
    return (a) => {
        console.log(this.a);
    };
}

var obj1 = {
    a: 2
};

var obj2 = {
    a: 3
};

a = 4;

var bar = foo.call(obj1);
bar.call(obj2);//2

var baz = foo.call(obj2);
baz.call(obj1);//3

foo()();//4
```

这就很明显了，外层的闭包封闭的是哪个作用域，箭头函数里的`this`指向的是就是哪个作用域。

> 我觉得有一点需要明确，程序里是没有玄学问题的，只不过是暂时没有发现原因，或者是了解的东西不够。