---
title: Webpack原理浅析之构建流程梳理
date: 2019-12-13 11:36:45
tags:
---


本文将带你梳理Webpack在构建过程和基本的构建原理，以及我们熟悉的loader,plugin都是怎样应用在构建流程中的。

## 一.概念前置
Tapable  
Tapable是个专注一实现自定义事件的发布与订阅的一个小型库。webpack中许多重要的对象都是继承自Tapable，以便于钩入到构建过程中的的各个关键节点做出处理。可以说Tapable是webpack整个插件系统实现的基础。

Compiler  
Compiler模块是 webpack 的支柱引擎，它通过 CLI 或 Node API 传递的所有选项，创建出一个 compilation 实例。它扩展(extend)自 Tapable 类，以便注册和调用插件。大多数面向用户的插件首，会先在 Compiler 上注册。

Compilation  
Compilation模块会被 Compiler 用来创建新的编译（或新的构建）。  compilation 实例能够访问所有的模块和它们的依赖（大部分是循环依赖）。它会对应用程序的依赖图中所有模块进行字面上的编译(literal compilation)。在编译阶段，模块会被加载(loaded)、封存(sealed)、优化(optimized)、分块(chunked)、哈希(hashed)和重新创建(restored)。

## 二.构建流程
初始化阶段
初始化参数
从配置文件和 Shell 语句中读取与合并参数，得出最终的参数。
在这个阶段也会对合并出的配置参数进校验，校验不通过，webpack将抛出错误。

```
// webpack.js

// 校验配置
	const webpackOptionsValidationErrors = validateSchema(
		webpackOptionsSchema,
		options
	);
	if (webpackOptionsValidationErrors.length) {
		throw new WebpackOptionsValidationError(webpackOptionsValidationErrors);
	}
```
实例化Compiler  
用上一步得到的参数初始化 Compiler 实例。Compiler 实例中包含了完整的 Webpack 配置，全局只有一个 Compiler 实例。
加载插件，配置文件中配置的插件在此时进行注册。已监听在构建过程中广播的事件。
内置插件的注册。
启动编译。
```
// 实例化compiler对象
	let compiler;
	if (Array.isArray(options)) {
		compiler = new MultiCompiler(options.map(options => webpack(options)));
	} else if (typeof options === "object") {
		options = new WebpackOptionsDefaulter().process(options);

		compiler = new Compiler(options.context);
		compiler.options = options;
		new NodeEnvironmentPlugin().apply(compiler);
		if (options.plugins && Array.isArray(options.plugins)) {
			for (const plugin of options.plugins) {
				// 注册配置插件的监听器 让插件可以监听到构建过程中广播的事件 并传入compiler
				if (typeof plugin === "function") {
					plugin.call(compiler, compiler);
				} else {
					plugin.apply(compiler);
				}
			}
		}
		compiler.hooks.environment.call();
		compiler.hooks.afterEnvironment.call();
		compiler.options = new WebpackOptionsApply().process(options, compiler);
	} else {
		throw new Error("Invalid argument: options");
	}
	if (callback) {
		if (typeof callback !== "function") {
			throw new Error("Invalid argument: callback");
		}
		// 监听文件变更
		if (
			options.watch === true ||
			(Array.isArray(options) && options.some(o => o.watch))
		) {
			const watchOptions = Array.isArray(options)
				? options.map(o => o.watchOptions || {})
				: options.watchOptions || {};
			return compiler.watch(watchOptions, callback);
		}
		// 开始编译
		compiler.run(callback);
	}
```    
编译阶段  
这个阶段主要是Compile创建出一个Compilation实例，开始一次编译。在Compile上主要会广播出一下事件节点。

run：启动一次新的编译。  
watch-run：和 run 类似，区别在于它是在监听模式下启动的编译，在这个事件中可以获取到是哪些文件发生了变化导致重新启动一次新的编译。  
compile：该事件是为了告诉插件一次新的编译将要启动，同时会给插件带上 compiler 对象。  
compilation： 当 Webpack 以开发模式运行时，每当检测到文件变化，一次新的 Compilation 将被创建。一个 Compilation 对象包含了当前的模块资源、编译生成资源、变化的文件等。Compilation 对象也提供了很多事件回调供插件做扩展。
make：一个新的 Compilation 创建完毕，即将从 Entry 开始读取文件，根据文件类型和配置的 Loader 对文件进行编译，编译完后再找出该文件依赖的文件，递归的编译和解析。  
after-compile：一次 Compilation 执行完成。  
invalid：当遇到文件不存在、文件编译错误等异常时会触发该事件，该事件不会导致 Webpack 退出。  
Compile是存在与整个webpack的生命周期当中，负责全局的调度。真正的一次编译的执行，是由Compilation进行的，Compilation阶段包含的事件有：  

build-module：使用对应的 Loader 去转换一个模块。  
normal-module-loader：在用 Loader 对一个模块转换完后，使用 acorn 解析转换后的内容，输出对应的抽象语法树（AST），以方便 Webpack 后面对代码的分析。
program：从配置的入口模块开始，分析其 AST，当遇到 require 等导入其它模块语句时，便将其加入到依赖的模块列表，同时对新找出的依赖模块递归分析，最终搞清所有模块的依赖关系。  
seal： 所有模块及其依赖的模块都通过 Loader 转换完成后，根据依赖关系开始生成 Chunk。  
输出阶段  
这一阶段可以理解为一次编译结束，Compilation的任务结束，所以控制权又交到了Compile。  

shouldEmit：所有需要输出的文件已经生成好，询问插件哪些文件需要输出，哪些不需要。  
emit： 确定好要输出哪些文件后，执行文件输出，可以在这里获取和修改输出内容。
afterEmit： 文件输出完毕。  
done： 成功完成一次完成的编译和输出流程。  
failed：如果在编译和输出流程中遇到异常导致 Webpack 退出时，就会直接跳转到本步骤，插件可以在本事件中获取到具体的错误原因。  
## 三.输出结果分析
[https://github.com/ReedSun/analysis-bundle.js-of-Webpack/blob/master/dist/bundle.js]()

## 四.如何写一个loader
1. loader单一职责
loader用于将各种类型的文件转换为Webpack能识别的文件类型，也是webpack敢号称‘打包一切’的原因。
loader的关键在于职责单一，只完成一种转换，这样才有助与我们组合各种loader得到我们想要的转换结果过。比如一处理scss文件为例，

代码会先交给 sass-loader 把 SCSS 转换成 CSS
把 sass-loader 输出的 CSS 交给 css-loader 处理，找出 CSS 中依赖的资源、压缩 CSS 等；
把 css-loader 输出的 CSS 交给 style-loader 处理，转换成通过脚本加载的 JavaScript 代码；
2. loader本质是一个npm包
loader本质是node模块，这个模块需要导出一个函数。在开发loader时，你只需关心输入和输出。

```
module.exports = function(source) {
 // source 为 compiler 传递给 Loader 的一个文件的原内容
 // ....
 return result;
}
由于 Loader 运行在 Node.js 中，你可以调用任何 Node.js 自带的 API，或者安装第三方模块进行调用.

const sass = require('node-sass');
module.exports = function(source) {
  return sass(source);
};
```
3.本地调试loader  
在开发 Loader 的过程中，为了测试编写的 Loader 是否能正常工作，需要把它配置到 Webpack 中后，才可能会调用该 Loader。 在开发过程中使用的 Loader 都是通过 Npm 安装的，要使用 Loader 时会直接使用 Loader 的名称。
如果还采取以上的方法去使用本地开发的 Loader 将会很麻烦，因为你需要确保编写的 Loader 的源码是在 node_modules 目录下。 为此你需要先把编写的 Loader 发布到 Npm 仓库后再安装到本地项目使用。
解决以上问题的便捷方法有两种，分别如下：

Npm link：Npm link 专门用于开发和调试本地 Npm 模块，能做到在不发布模块的情况下，把本地的一个正在开发的模块的源码链接到项目的 node_modules 目录下，让项目可以直接使用本地的 Npm 模块。
ResolveLoader：ResolveLoader 用于配置 Webpack 如何寻找 Loader。 默认情况下只会去 node_modules 目录下寻找，为了让 Webpack 加载放在本地项目中的 Loader 需要修改 resolveLoader.modules。
五.如何写一个plugin
在 Webpack 运行的生命周期中会广播出许多事件，Plugin 可以监听这些事件，在合适的时机通过 Webpack 提供的 API 改变输出结果。

```
// 一个基础的plugin

    class BasicPlugin{
        // 在构造函数中获取用户给该插件传入的配置
        constructor(options){
        }

        // Webpack 会调用 BasicPlugin 实例的 apply 方法给插件实例传入 compiler 对象
        apply(compiler){
            compiler.plugin('compilation',function(compilation) {
                // 在插件中compiler和compilation我们都可以访问的到

            })
        }
    }

    // 导出 Plugin
    module.exports = BasicPlugin;
事件流机制
Compiler 和 Compilation 都继承自 Tapable，可以直接在 Compiler 和 Compilation 对象上广播和监听事件，方法如下：
/**
* 广播出事件
* event-name 为事件名称，注意不要和现有的事件重名
* params 为附带的参数
*/
compiler.apply('event-name',params);

/**
* 监听名称为 event-name 的事件，当 event-name 事件发生时，函数就会被执行。
* 同时函数中的 params 参数为广播事件时附带的参数。
*/
compiler.plugin('event-name',function(params) {

});
```
在开发插件时，还需要注意以下几点：  
只要能拿到 Compiler 或 Compilation 对象，就能广播出新的事件，所以在新开发的插件中也能广播出事件，给其它插件监听使用。  
传给每个插件的 Compiler 和 Compilation 对象都是同一个引用。也就是说在一个插件中修改了 Compiler 或 Compilation 对象上的属性，会影响到后面的插件。
有些事件是异步的，这些异步的事件会附带两个参数，第二个参数为回调函数，在插件处理完任务时需要调用回调函数通知 Webpack，才会进入下一处理流程。例如：
```
compiler.plugin('emit',function(compilation, callback) {
   // 支持处理逻辑

   // 处理完毕后执行 callback 以通知 Webpack 
   // 如果不执行 callback，运行流程将会一直卡在这不往下执行 
   callback();
 });
```
 [https://juejin.im/entry/5b0e3eba5188251534379615]()



