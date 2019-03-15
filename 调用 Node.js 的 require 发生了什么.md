### hello world code

```js
// printConsole.js

module.exports = function (content) {
    console.log(content);
}
```

```js
// index.js

const print = require('./printConsole.js');

print('hello world');
```

我们创建了一个名为 `printConsole.js` 的文件，暴露了一个调用了 `console.log` 的函数，在 `index.js` 中，我们通过 `require('./printConsole.js')` 来引用该模块，那为什么可以用 `require` 呢？我们通过源码分析来探索下 `node.js` 的实现。

### require 的 js 部分

#### lib/module.js

只有一句，`module.exports = require('internal/modules/cjs/loader.js')`。

打开 `loader.js` 可以看到真正的 `loader` 实现部分。

#### lib/internal/modules/cjs/loader.js

##### Module.prototype.require

对字符串参数进行简单的校验，内部直接调用了内部方法 `Module._load()`。

##### Module._load

根据传入的模块名，首先从 Cache 中查询是否已经加载过该模块了，如果已经有缓存了，则直接调用缓存的 module 的 `exports` 方法，返回该值，这里也就对应了我们在代码中，想要将一个模块暴露出去给第三方使用时，使用 `module.exports = {...}` 的形式来实现。

第二步，判断该模块是 `NativeModule` 还是普通的 `3rd-party` 模块，因为两种模块的加载方式不同，这里我们先分析第三方的模块。

根据模块的文件名，调用 `new Module(fileName, parent)` 生成一个 Module 对象，然后放在 Modules 的缓存池中，调用 `tryModuleLoad(module, fileName)`。

##### tryModuleLoad

该函数有抛异常的可能，内部调用了 `module.load(fileName)`，如果出现异常，则从缓存池中将之前一步已经缓存的 `module` 移除。

##### Module.prototype.load

在 `load` 函数中，根据 `fileName` 查询到完整的 `path`，例如如果通过包管理器引入的依赖，我们在代码中 `require()` 是不需要写出完整路径的，但是 node 能够成功从本地或全局的 `node_modules` 文件夹中找到对应的模块，这个操作即是在 `Module.prototype.load` 中被完成的。

在得到模块的完整路径后，会尝试解析该路径的后缀（例如 `.js`，`.json` 等等），这里的取后缀会准确找到最长的路径，因为我们的文件名中有可能会有多个 `.` ，例如在 React Native 工程中很常见的使用 `index.ios.js` 和 `index.android.js` 来区分不同平台，而在 `load` 函数中，将只取到 `.js` 的后缀。

取到后缀的作用是什么呢？这里就需要介绍 Node 提前预定义的几个对后缀处理的函数，这里我们先聚焦在处理 `.js` 后缀的函数上。

##### Module._extensions['.js']

定位到 `Module._extensions['.js']` 上，可以发现这是一个很简单的函数，包含两句，首先根据之前已经解析出来的完整路径，以 `utf-8` 的编码调用 `fs.readFileSync(fileName, 'utf8')`，得到 `.js` 文件的内容（也即是字符串）。

> 为什么我们在这里就已经可以通过 `require('fs')` 的方式来使用 `fs.readFileSync` 了呢？这是因为 `fs` 模块是 `node` 的内建模块，会在这之前已经被加载完成。

上一句得到的结果，最终会被传给 `module._compile()` 函数。

##### Module.prototype._compile

`Module.prototype._compile` 是在简化的 `require` 流程上， JavaScript 端处理的最后一环，忽略掉一些判断逻辑，我们来看看这个函数内部做了些什么。

首先，调用了 `Module.wrap(jsContent)` 函数，而这个函数的实现也非常简单，仅仅是将 `'(function (exports, require, module, __filename, __dirname) { '` 和 `'\n});'` 两个字符串拼接在传入的 `jsContent` 头部和尾部，构成一个函数的字符串。

得到的函数字符串被传递给 `vm.runInThisContext` 函数。

##### vm.runInThisContext

因为这里涉及到 Node 的 `Context`，`Handle`，`Scope` 等概念，而详细解释这些不是这篇文章的重点，所以我们这里可以先简单的认为 `Context` 是代码执行的上下文。

我们跳转到 `vm.js` 中可以看到，`module.exports` 中暴露的 `runInThisContext` 函数的内部首先根据传入的 `code` （也即是之前被拼接了的函数的字符串），创建 `Script` 对象，之后对该 `Script` 对象调用 `runInThisContext`。

`Script` 对象继承于 `ContextifyScript`，而 `ContextifyScript` 并不是 JavaScript 的模块，而是通过调用 `internalBinding()` 直接绑定到 Node 的 C++ 类。所以这里我们调用 `scriptInstance.runInThisContext()` 实际上会被传递到 C++ 类上，而该函数是将之前 `wrap` 过的函数字符串转换为可执行函数，由于 JS 与 C++ 的互相调用需要了解 `Context`，`Scope`，`Handle` 等概念，所以这里我们可以先简单的认为 `runInThisContext()` 的作用是在当前作用域下 `eval` 一个可执行的函数，执行完成后得到结果。最终回到 `_load` 函数，返回 `module.exports`。

至此，完成了一次对第三方模块的 `require` 引用。