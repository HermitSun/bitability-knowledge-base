### 再谈JavaScript中的this

之前一篇文章里我曾经谈过`this`的事情，当时觉得那几种常规的绑定方式没啥可说的，当时的着眼点是Node环境，然而用jQuery的时候（要恰饭的嘛）遇到了一些很有意思的事情，觉得在浏览器环境中的使用也还是有必要一说的，就决定再写一篇。

直接上代码吧（导包的过程就略去不表了），当然，也不要太在意这种老式写法：


```html
<div>
    <ul>
        <li id="one"><em>fresh</em> figs</li>
        <li id="two">pine nuts</li>
        <li id="three">honey</li>
        <li id="four">balsamic vinegar</li>
    </ul>
</div>
<script>
    $(function () {
        $('li').each(function () {
            $(this).append('<em>' + this.id + '</em>');
        }); 
    });
</script>
```

之所以没用模板字符串，是因为IE不支持ES6语法，会直接报错……这都IE11了还不支持。所以，后面关于箭头函数中的`this`问题，以及涉及到ES6语法的部分，就全部在现代浏览器（比如Chrome）环境下探讨了。

这段代码的含义很明确，就是获取每一个`<li>`里的id，然后用斜体加在它的内容的后面。这里的`this`指向的是循环中的每一个`<li>`。至于为什么用`$(this)`，无非是为了创建一个jQuery的对象，这样可以调用jQuery的相关方法。正常效果是这样的（虽然确实丑了一点）：

![](D:\BitEnergyProject\项目组材料\孙文撰写的材料\jquery1.png)

不过，要是把`$(this)`外面的`$`去掉，会发生什么？也就是说，此时的JavaScript代码是这样的：

```javascript
$(function () {
    $('li').each(function () {
        $(this).append('<em>' + this.id + '</em>');
    }); 
});
```

效果是这样的（Chrome），明显不对；后面的标签似乎被直接当成字符串了：

![](D:\BitEnergyProject\项目组材料\孙文撰写的材料\jquery2.png)

而在IE里是这样的，直接连后面的内容都没有了：

![](D:\BitEnergyProject\项目组材料\孙文撰写的材料\jquery3.png)

这是为什么？按照习惯，肯定是先看控制台。Chrome的控制台风平浪静，而IE的控制台里很无情地报错了：

`SCRIPT438: 对象不支持“append”属性或方法`

那么，问题就出在这个append方法上了。查一下MDN的文档，可以看到有这样一个方法：

>The **ParentNode.append()** method inserts a set of `Node`objects or `DOMString` objects after the last child of the `ParentNode`. `DOMString` objects are inserted as equivalent `Text` nodes.

而这个方法是不兼容IE的。所以，这里的`this`是被当成`Node`对象处理的，调用了这个方法把字符串写到了每个`<li>`里。其实这次的问题跟上一篇文章里的那个`values`异曲同工。

不过，这里的`this`为什么会是`Node`对象？可能这个问题问得有点蠢，因为按照常规思维来说，由谁调用，`this`就应该指向谁；毕竟，调用者应当是持有被调用者的。大部分语言确实也是这么做的；对于JavaScript来说，隐式绑定也是这么实现的，但是隐式绑定存在一个“隐式绑定丢失”的问题，比如下面这段代码，输出可能会和预料中的不太一样：

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
bar(); // 3
```

所以问题没这么简单，还是有必要从头开始说起。`this`一共有四种绑定方式，分别是默认绑定、隐式绑定、显式绑定和new 绑定。优先级是从右往左依次递减的。事实上，问题主要也就是出在前两种绑定里面。

其实，分析`this`的指向，主要还是分析函数的调用栈。栈顶是当前的函数，而第二个是`this`的指向，也就是当前的调用环境。不过这种方法有时候确实比较麻烦，也有一些类似于规则的东西，比如：

1. 无法应用其他规则时一定是默认绑定。

2. 独立函数调用和匿名函数调用一定指向全局。

3. 至于隐式绑定，一层调用（由对象直接调用）是指向调用者，超过两层就变成默认绑定（所谓的绑定丢失）。这个也好理解，就类似于一个二级指针，最终的指向还是被调用者本身，需要在被调用者的环境里分析。

   ……

之前那段代码，相当于是隐式绑定，参数里那个匿名函数的运行环境（调用栈的第二层）是`<li>`，也就是说，`this`指向的是每一个`<li>`，也就有了接下来的一系列操作。

如果换成箭头函数呢？也就是说，如果把代码改成这样：

```javascript
$(function () {
    $('li').each(() => {
        $(this).append('<em>' + this.id + '</em>');
    }); 
});
// 有'$': Uncaught TypeError: Cannot read property 'createDocumentFragment' of null
// 无'$': Uncaught DOMException: Failed to execute 'append' on 'Document'
```

其实加不加`$`都是一样报错，只不过报错的内容不太一样。从不加`$`的报错可以看出，此时的`this`是指向`document`的。

因为箭头函数是利用作用域（而不是调用栈，完全改变了`this`的机制）实现的`this`，所以这里的`this`按照作用域指向了`document`。作用域的问题我之前也谈过，这里因为是在`$(function(){})`的内部，作用域是`document`（其实是`$(document).ready(function(){})`的简写，还是隐式绑定），所以`this`指向的就是`document`。

可以预料，如果把最外层也改成箭头函数，这里的`this`就会指向全局（浏览器里是`window`，Node环境下会有区别）。事实也的确如此：

```javascript
$(() => {
    console.log(this);// window
    $('li').each(function () {
        $(this).append('<em>' + this.id + '</em>');
    });
});
```

当然，下面的语句是不受影响的，因为机制完全不同。箭头函数的机制如同“六亲不认”，完全不考虑调用与被调用，只看自己所在的位置，然后根据词法作用域决定自己的指向；而普通的`this`机制就需要考虑调用栈以及一些特殊的规则；说是特殊，原理上还是分析调用栈。

当然，最佳实践应当是避免混用这两种风格。