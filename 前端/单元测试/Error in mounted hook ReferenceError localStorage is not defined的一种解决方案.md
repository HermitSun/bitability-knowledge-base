### Error in mounted hook: "ReferenceError: localStorage is not defined"的解决方案

先说说问题背景。首先Vue肯定是没跑了，在这个项目中，我没有使用传统（也更加熟悉）的JavaScript，而是使用了TypeScript，同时引入了Mocha+Chai，结合官方推出的Vue Test Utils进行单元测试。Vue Test Utils好用是好用，但可能是因为还没有完全成熟，坑还是挺多的。回头写一篇介绍一下我的一些踩坑经验。

这次遇到的问题是，在测试页面（也就是`.vue`文件）的时候，使用了Vue Test Utils的mount/shallowMount方法进行页面的创建。因为页面里用到了localStorage来进行一些用户数据的存取，页面始终无法挂载，报错`Error in mounted hook: "ReferenceError: localStorage is not defined`。

实话说看着满屏飘红有点闹心……不过这个报错倒也正常，理论上页面上调用的localStorage是`window.localStorage`，而单元测试显然是运行在Node环境下的，Node环境里没有window这个全局变量（取而代之的是global，我在[之前的文章](https://blog.csdn.net/HermitSun/article/details/86626076)里提到过），自然也没有`window.localStorage`。vue-cli里集成的单元测试插件`@vue/cli-plugin-unit-mocha`的文档里也明确指出了这一点：

> **vue-cli-service test:unit**
>
> Run unit tests with mocha-webpack + chai.
>
> **Note the tests are run inside Node.js with browser environment simulated with JSDOM.**

但这里的环境还有点区别，要说完全没有`window`也似乎不太正确，因为这里还引入了JSDOM。JSDOM这个包是用来在Node环境中模拟浏览器环境中的`window`的。具体的使用细节可以自行查阅文档，但只要明确这里是有window的就行了。

理论上，我们只要往window里添加一个localStorage就可以解决问题。首先说明，下文都是基于JS的讨论，因为这些库的实现都是JS。所以常规思路肯定是类似这样的实现：

```javascript
window.localStorage = {
    data: {},
    setItem(key, value) {
        this.data[key] = value;
    },
    getItem(key) {
        return this.data[key];
    }
};
// TypeError: Cannot set property localStorage of #<Window> which has only a getter
```

这种写法，在非严格模式下会静默失败，而如果在严格模式下，就会报错。这也是很有意思的一件事，因为这里的`window.localStorage`只有getter，是不能赋值的。所以，必须要迂回一下。

后来在尝试中，我得出了一个解决方案：首先引入`dom-storage`这个包，`dom-storage`其实就是实现了localStorage的功能，顺便提供了一些文件读写的功能。然后创建一个js文件，比如就叫`localStorage.js`吧：

```javascript
// localStorage.js
const Storage = require('dom-storage');
global.localStorage = new Storage(null, {strict: true});
window.localStorage = global.localStorage;
```

然后在单元测试中引入这个文件。问题解决。

欣喜之余，仔细思考一下这段代码，就会发现蹊跷：`window.localStorage`不是不能赋值吗？将代码改动一下，立刻出现了一样的报错：

```javascript
// localStorage.js
'use strict';
const Storage = require('dom-storage');
global.localStorage = new Storage(null, {strict: true});
window.localStorage = global.localStorage;
// TypeError: Cannot set property localStorage of #<Window> which has only a getter
```

结论说起来也简单：进行单元测试时，测试逻辑调用的应该是`global.localStorage`。为了验证这一点，在页面上加上console.log，输出果然如此：

```javascript
// Test.vue
mounted() {
    console.log(localStorage);
}

// localStorage.js 
const Storage = require('dom-storage');
global.localStorage = new Storage(null, {strict: true});
window.localStorage = global.localStorage;
window.localStorage.setItem('test', 'test');
console.log(window.localStorage);
// Storage { test: 'test' }
// Storage {}
```

其实，最后还是回到了文档上的那句话：

> **Note the tests are run inside Node.js with browser environment simulated with JSDOM.**

只要是运行在Node环境中，就免不了和global交互。绕了这么大一个圈子，主要是JSDOM给人带来的误会太深，让人以为真的在Node中创建了一个window，并且能够代替global；实则不然。

最后提一下，这个文件用JS写比较省事。因为TS会想方设法阻挠我们修改`global`，为了解决这个小问题而去自行扩展.d.ts似乎有点得不偿失。虽然往TS里混入了JS不太好，不过能解决问题，也没必要在意这些细节了。

当然，非要写也不是不行，但是得先扩展原有的接口：

```typescript
// shims-node.d.ts
declare namespace NodeJS {
    interface Global {
        localStorage: any
    }
}

// localStorage.ts
global.localStorage = {
    data: {},
    getItem(key: string) {
        return this.data[key];
    },
    setItem(key: string, value: string) {
        this.data[key] = value;
    }
};
```

结束。

------

这次为了写单元测试，遇到的奇怪的bug多了之后，看到这些似乎也有点麻木了……

众所周知，vue-cli3比vue-cli2要好用不少，更加精简的项目结构，更快的构建速度，高度抽象的配置文件，更多契合度更高的插件，更加平滑的开箱体验，都让人感觉很舒服。说完好处，也说说坏处，vue-cli3的简洁其实是一把双刃剑，在方便的同时也带来了难以发现隐藏bug的问题。实在是难受。

在此之前，我还试验了其他几种方案，但都以失败告终，出现了各种奇怪的问题。其中最奇怪的一个，是在使用`mock-local-storage`的时候出现的。引入这个包之后（vue-cli3创建项目时，如果选择了单元测试，会自带`@vue/cli-plugin-unit-mocha`；而这个包里面会引入`jsdom-global`），通过`vue-cli-service test:unit --require jsdom-global --require mock-local-storage`调用会报错，还是找不到localStorage；而如果换成`vue-cli-service test:unit --require jsdom-global mock-local-storage`，则会变成这样：

```powershell
WEBPACK  Compiled successfully in xxxxms
MOCHA  Testing...
0 passing (0ms)
MOCHA  Tests completed successfully
```

测试看似通过了，但实际上根本没有进行。顺着调用栈一路跟踪下去，在项目根目录的`/node_modules/@vue/cli-plugin-unit-mocha/index.js`的第60行（可能不同的项目/系统/IDE中显示会有偏差，但就是这一行附近）发现了问题，这一行中的`hasInlineFilesGlob`会被赋值为true，导致后续的测试逻辑直接被跳过。原因是命令中`--require`后面跟着超过一个模块时会触发这一段逻辑。但为什么会这样，我还不清楚；主要是经验尚浅，对webpack不了解。因此姑且把问题记录下来，以后再慢慢探索。如果有人知道，请不吝赐教。