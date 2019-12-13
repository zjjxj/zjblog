---
title: webpack原理浅析之代码分割
date: 2019-12-13 11:37:54
tags:
---
code splitting是webpack最引人注目的特性之一。他可以把代码分离到不同的bundle中，然后可以按需加载或并行加载这些文件。

## 一.可以实现代码分割的方式
### 1.入口起点：使用 entry 配置手动地分离代码。
这是迄今为止最简单、最直观的分离代码的方式。不过，这种方式手动配置较多，并有一些隐患：

如果入口 chunk 之间包含一些重复的模块，那些重复模块都会被引入到各个 bundle 中。
这种方法不够灵活，并且不能动态地将核心应用程序逻辑中的代码拆分出来。
### 2.SplitChunksPlugin
SplitChunksPlugin 插件可以将公共的依赖模块提取到已有的 entry chunk 中，或者提取到一个新生成的 chunk。

### 3.动态导入(懒加载或者按需加载)
当涉及到动态代码拆分时，webpack 提供了两个类似的技术。第一种，也是推荐选择的方式是，使用符合 ECMAScript 提案 的 import() 语法 来实现动态导入。第二种，则是 webpack 的遗留功能，使用 webpack 特定的 require.ensure。

## 二.SplitChunksPlugin
Webpack4.0以后，我们可谓是告别了头疼的配置问题。Webpack引入了mode的配置选项，提供’production’,’development’和’none’三种模式选择，告知 webpack 使用相应环境的内置优化。

当我们开启不同的模式，就意味着载入了相对的默认配置。默认情况下，mode的值为’production’。 我们可以去官网看看production模式下都有哪些默认配置。

production模式下，SplitChunksPlugin插件是默认被启用的，默认配置如下
```
module.exports = {
    //...
    optimization: {
      splitChunks: {
        chunks: 'async',
        minSize: 30000, // 打包出的chunk最小时30kb 否则不打包
        maxSize: 0,
        minChunks: 1, // 至少被1个module引用
        maxAsyncRequests: 5,  //  
        maxInitialRequests: 3, // 初始化时最多三个请求
        automaticNameDelimiter: '~', // 名字中间的分隔符
        name: true, // chunk的名字，如果设成true，会根据被提取的chunk自动生成
        cacheGroups: {
          vendors: {
            test: /[\\/]node_modules[\\/]/, // 文件匹配规则
            priority: -10 // 优先级
          },
          default: {
            minChunks: 2,
            priority: -20,
            reuseExistingChunk: true // 当module未变时，是否可以使用之前的chunk
          }
        }
      }
    }
};
```
一些重要的配置项：

chunks——表示哪些代码需要优化 有三个可选值：initial(初始块)、async(按需加载块)、all(全部块)，默认为async。  
initial, all模式 会将所有来自node_modules的模块分配到一个叫vendors的缓存组；所有重复引用至少两次的代码，会被分配到default的缓存组。  
initial模式下会分开优化打包异步和非异步模块。而all会把异步和非异步同时进行优化打包。    
cacheGroups——可以自定义配置打包块  
webpack默认的打包chunk优化，只会影响按需加载的模块，也就是说只有异步加载的模块才会有默认打包行为。（import().then()，require.ensure，react-router按需加载）

## 三.Code Splitting实现原理
首先我们依然创建一个简单入口模块index.js和两个依赖模块foo.js和bar.js
```
// index.js
'use strict';
import(/* webpackChunkName: "foo" */ './foo').then(foo => {
    console.log(foo());
})
import(/* webpackChunkName: "bar" */ './bar').then(bar => {
    console.log(bar());
})
// foo.js
'use strict';
exports.foo = function () {
    return 2;
}
// bar.js
'use strict';
exports.bar = function () {
    return 1;
}
```
webpack配置如下
```
var path = require("path");

module.exports = {
    entry: path.join(__dirname, 'index.js'),
    output: {
        path: path.join(__dirname, 'outs'),
        filename: 'index.js',
        chunkFilename: '[name].bundle.js'
    },
};
```
打包结果(去掉了部分注释)
```
(function(modules) { // webpackBootstrap
    // install a JSONP callback for chunk loading
    var parentJsonpFunction = window["webpackJsonp"];
    window["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules, executeModules) {
        // add "moreModules" to the modules object,
        // then flag all "chunkIds" as loaded and fire callback
        var moduleId, chunkId, i = 0, resolves = [], result;
        for(;i < chunkIds.length; i++) {
            chunkId = chunkIds[i];
            if(installedChunks[chunkId]) {
                resolves.push(installedChunks[chunkId][0]);
            }
            installedChunks[chunkId] = 0;
        }
        for(moduleId in moreModules) {
            if(Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
                modules[moduleId] = moreModules[moduleId];
            }
        }
        if(parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules, executeModules);
        while(resolves.length) {
            resolves.shift()();
        }
    };
    // The module cache
    var installedModules = {};
    // objects to store loaded and loading chunks
    var installedChunks = {
        2: 0
    };
    // The require function
    function __webpack_require__(moduleId) {
        // Check if module is in cache
        if(installedModules[moduleId]) {
            return installedModules[moduleId].exports;
        }
        // Create a new module (and put it into the cache)
        var module = installedModules[moduleId] = {
            i: moduleId,
            l: false,
            exports: {}
        };
        // Execute the module function
        modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
        // Flag the module as loaded
        module.l = true;
        // Return the exports of the module
        return module.exports;
    }
    // This file contains only the entry chunk.
    // The chunk loading function for additional chunks
    __webpack_require__.e = function requireEnsure(chunkId) {
        var installedChunkData = installedChunks[chunkId];
        if(installedChunkData === 0) {
            return new Promise(function(resolve) { resolve(); });
        }
        // a Promise means "currently loading".
        if(installedChunkData) {
            return installedChunkData[2];
        }
        // setup Promise in chunk cache
        var promise = new Promise(function(resolve, reject) {
            installedChunkData = installedChunks[chunkId] = [resolve, reject];
        });
        installedChunkData[2] = promise;
        // start chunk loading
        var head = document.getElementsByTagName('head')[0];
        var script = document.createElement('script');
        script.type = 'text/javascript';
        script.charset = 'utf-8';
        script.async = true;
        script.timeout = 120000;
        if (__webpack_require__.nc) {
            script.setAttribute("nonce", __webpack_require__.nc);
        }
        script.src = __webpack_require__.p + "" + ({"0":"foo","1":"bar"}[chunkId]||chunkId) + ".bundle.js";
        var timeout = setTimeout(onScriptComplete, 120000);
        script.onerror = script.onload = onScriptComplete;
        function onScriptComplete() {
            // avoid mem leaks in IE.
            script.onerror = script.onload = null;
            clearTimeout(timeout);
            var chunk = installedChunks[chunkId];
            if(chunk !== 0) {
                if(chunk) {
                    chunk[1](new Error('Loading chunk ' + chunkId + ' failed.'));
                }
                installedChunks[chunkId] = undefined;
            }
        };
        head.appendChild(script);
        return promise;
    };
    // expose the modules object (__webpack_modules__)
    __webpack_require__.m = modules;
    // expose the module cache
    __webpack_require__.c = installedModules;
    // define getter function for harmony exports
    __webpack_require__.d = function(exports, name, getter) {
        if(!__webpack_require__.o(exports, name)) {
            Object.defineProperty(exports, name, {
                configurable: false,
                enumerable: true,
                get: getter
            });
        }
    };
    // getDefaultExport function for compatibility with non-harmony modules
    __webpack_require__.n = function(module) {
        var getter = module && module.__esModule ?
            function getDefault() { return module['default']; } :
            function getModuleExports() { return module; };
        __webpack_require__.d(getter, 'a', getter);
        return getter;
    };
    // Object.prototype.hasOwnProperty.call
    __webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };
    // __webpack_public_path__
    __webpack_require__.p = "";
    // on error function for async loading
    __webpack_require__.oe = function(err) { console.error(err); throw err; };
    // Load entry module and return exports
    return __webpack_require__(__webpack_require__.s = 0);
})
([
(function(module, exports, __webpack_require__) {
    "use strict";
    __webpack_require__.e/* import() */(0).then(__webpack_require__.bind(null, 1)).then(foo => {
        console.log(foo());
    })
    __webpack_require__.e/* import() */(1).then(__webpack_require__.bind(null, 2)).then(bar => {
        console.log(bar());
    })
})
]);
```
可以看出import()返回的是一个Promise  
通过动态脚本的注入实现异步加载模块。  
再通过webpackJsonp作为模块加载和执行完成的回调，从而触发import的resolve。
