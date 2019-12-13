---
title: webpack原理浅析之打包优化
date: 2019-12-13 11:38:31
tags:
---
## 一. DLL(动态链接库)
### 1.为什么需要dll
Webpack的打包构建速度会随着引入的库的数量增加而变的越来越慢，一些体积比较大的第三方库是构建速度慢的主要原因。  
虽然我们有code split方案，但是代码分割后每次打包还是要去处理这些第三方库。
在开发中，每次我们修改源代码，连带着没有改动的第三方库也会被重新编译打包，这是我们不想看到的。

DLL就是Webpack提供的专门为我们解决这个问题的神器。

DllPlugin结合DllRefrencePlugin插件的运用，对将要产出的bundle文件进行拆解打包，可以很彻底地加快webpack的打包速度，从而在开发过程中极大地缩减构建时间。

其主要原理是： DLLPlugin 它能把第三方库代码分离开，并且每次文件更改的时候，它只会打包该项目自身的代码。所以打包速度会更快。

### 2.如何使用
DLLPlugin 这个插件是在一个额外独立的webpack设置中创建一个只有dll的bundle，也就是说我们在项目根目录下除了有webpack.config.js，还会新建一个webpack.dll.config.js文件。  
webpack.dll.config.js作用是把所有的第三方库依赖打包到一个bundle的dll文件里面，还会生成一个名为 manifest.json文件。  
该manifest.json的作用是用来让 DllReferencePlugin 映射到相关的依赖上去的。

```

// webpack.dll.config.js


const webpack = require('webpack');
// 需要单独打包的第三方库
const vendors = [
  'antd',
  'isomorphic-fetch',
  'react',
  'react-dom',
  'react-redux',
  'react-router',
  'redux',
  'redux-promise-middleware',
  'redux-thunk',
  'superagent',
]; 

module.exports = {
  output: {
    path: 'build',
    filename: '[name].[chunkhash].js',
    library: '[name]_[chunkhash]',
  },
  entry: {
    vendor: vendors,
  },
  plugins: [
    new webpack.DllPlugin({
      path: 'manifest.json', // manifest文件的输出路径
      name: '[name]_[chunkhash]', // dll暴露的对象名 要跟output.library保持一致
      context: __dirname, // 解析包路径的上下文 ，这个要跟接下来配置的dll user一致
    }),
  ],
};
运行Webpack dll配置，会输出两个文件，一个是打包好的vendor.js，一个就是manifest.json.

// manifest.json
{
  "name": "vendor_ac51ba426d4f259b8b18",
  "content": {
    "./node_modules/antd/dist/antd.js": 1,
    "./node_modules/react/react.js": 2,
    "./node_modules/react/lib/React.js": 3,
    "./node_modules/react/node_modules/object-assign/index.js": 4,
    "./node_modules/react/lib/ReactChildren.js": 5,
    "./node_modules/react/lib/PooledClass.js": 6,
    "./node_modules/react/lib/reactProdInvariant.js": 7,
    "./node_modules/fbjs/lib/invariant.js": 8,
    "./node_modules/react/lib/ReactElement.js": 9,
    
    ............
Webpack将每个库都进行了编号索引，之后的dll user可以读取这个文件，直接用id来引用。

// webpack.config.js

const webpack = require('webpack');

module.exports = {
  output: {
    path: 'build',
    filename: '[name].[chunkhash].js',
  },
  entry: {
    app: './src/index.js',
  },
  plugins: [
    new webpack.DllReferencePlugin({
      context: __dirname, // 需要跟之前保持一致
      manifest: require('./manifest.json'),// 用来引入刚才输出的manifest文件
    }),
  ],
};

```
运行webpack，会发现生成的包和构建速度都有了一个质的提升。

## 二.HappyPack——提升构建速度
webpack需要处理的文件是非常多的，构建过程是一个涉及大量文件读写的过程。项目复杂起来了，文件数量变多之后，webpack构建就会特别慢。  
由于运行在 Node.js 之上的 Webpack 是单线程模型的，所以Webpack 需要处理的事情需要一件一件的做，不能多件事一起做。  
我们需要Webpack 能同一时间处理多个任务，发挥多核 CPU 电脑的威力，HappyPack 就能让 Webpack 做到这点，它把任务分解给多个子进程去并发的执行，子进程处理完后再把结果发送给主进程。