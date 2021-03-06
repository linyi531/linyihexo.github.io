---
title: webpack流程
date: 2020-04-17 10:29:41
tags:
  - webpack
categories: 前端工具
cover_img: https://tva1.sinaimg.cn/large/007S8ZIlgy1ggbpmleo2gj31900u0qv6.jpg
feature_img: https://tva1.sinaimg.cn/large/007S8ZIlgy1ggbpmleo2gj31900u0qv6.jpg
---

# webpack流程

[细说webapck流程](http://taobaofed.org/blog/2016/09/09/webpack-flow/)

[webpack原理](https://juejin.im/entry/5b0e3eba5188251534379615)

[ webpack 模块加载机制](https://juejin.im/post/5aa54608f265da23826d9b34)

流程概述：

Webpack 的运行流程是一个串行的过程，从启动到结束会依次执行以下流程：

1. 初始化参数：从配置文件和 Shell 语句中读取与合并参数，得出最终的参数；
2. 开始编译：用上一步得到的参数初始化 Compiler 对象，加载所有配置的插件，执行对象的 run 方法开始执行编译；
3. 确定入口：根据配置中的 entry 找出所有的入口文件；
4. 编译模块：从入口文件出发，调用所有配置的 Loader 对模块进行翻译，再找出该模块依赖的模块，再递归本步骤直到所有入口依赖的文件都经过了本步骤的处理；
5. 完成模块编译：在经过第4步使用 Loader 翻译完所有模块后，得到了每个模块被翻译后的最终内容以及它们之间的依赖关系；
6. 输出资源：根据入口和模块之间的依赖关系，组装成一个个包含多个模块的 Chunk，再把每个 Chunk 转换成一个单独的文件加入到输出列表，这步是可以修改输出内容的最后机会；
7. 输出完成：在确定好输出内容后，根据配置确定输出的路径和文件名，把文件内容写入到文件系统。

## webpack配置

**webpack.config.js**

```javascript
var path = require('path');
var node_modules = path.resolve(__dirname, 'node_modules');
var pathToReact = path.resolve(node_modules, 'react/dist/react.min.js');

module.exports = {
  // 入口文件，是模块构建的起点，同时每一个入口文件对应最后生成的一个 chunk。
  entry: {
    bundle: [
      'webpack/hot/dev-server',
      'webpack-dev-server/client?http://localhost:8080',
      path.resolve(__dirname, 'app/app.js')
    ]
  },
  // 文件路径指向(可加快打包过程)。
  resolve: {
    alias: {
      'react': pathToReact
    }
  },
  // 生成文件，是模块构建的终点，包括输出文件与输出路径。
  output: {
    path: path.resolve(__dirname, 'build'),
    filename: '[name].js'
  },
  // 这里配置了处理各模块的 loader ，包括 css 预处理 loader ，es6 编译 loader，图片处理 loader。
  module: {
    loaders: [
      {
        test: /\.js$/,
        loader: 'babel',
        query: {
          presets: ['es2015', 'react']
        }
      }
    ],
    noParse: [pathToReact]
  },
  // webpack 各插件对象，在 webpack 的事件流中执行对应的方法。
  plugins: [
    new webpack.HotModuleReplacementPlugin()
  ]
};
```

## 核心概念

- **loader**：能转换各类资源，并处理成对应模块的加载器。loader 间可以串行使用。

  loader引用顺序：loader从右往左执行，依次传递参数，有依赖的放在右边。

  loader 可以链式调用，上一个 loader 的输出被作为输入传给下一个 loader。 使用 [rule.use](https://www.webpackjs.com/configuration/module#rule-use) 配置选项，`use` 可以设置为一个 loader 数组。 在 webpack 1 中，loader 通常被用 `!` 连写。这一写法在 webpack 2 中只在使用旧的选项 `module.loaders` 时才有效。

  ```diff
    module: {
  -   loaders: [{
  +   rules: [{
        test: /\.less$/,
  -     loader: "style-loader!css-loader!less-loader"
  +     use: [
  +       "style-loader",  // 将 JS 字符串生成为 style 节点
  +       "css-loader",   // 将 CSS 转化成 CommonJS 模块
  +       "less-loader"   // 将 Sass 编译成 CSS
  +     ]
      }]
    }
  ```

  loader 可以配置以下参数:

  - test: 匹配处理文件的扩展名的正则表达式
  - use: loader名称
  - include/exclude: 手动指定必须处理的文件夹或屏蔽不需要处理的文件夹
  - query: 为loader提供额外的设置选项

  ```javascript
  module.exports = {
      //other code
      module: {
          rules: [
              {
                  test: /\.css/,
                  use: ['style-loader', 'css-loader'],
                  exclude: /node_modules/,
                  include: path.resolve(__dirname, 'src')
              }
          ]
      }
  }
  ```

- **chunk**：code splitting 后的产物，也就是按需加载的分块，装载了不同的 module。

  对于 module 和 chunk 的关系可以参照 webpack 官方的这张图：

  ![image-20190721095459839](https://tva1.sinaimg.cn/large/006y8mN6ly1g6j5wen5koj314r0u0k6k.jpg)

- plugin：webpack 的插件实体，这里以 UglifyJsPlugin 为例。

  ```javascript
  function UglifyJsPlugin(options) {
    this.options = options;
  }
  
  module.exports = UglifyJsPlugin;
  
  UglifyJsPlugin.prototype.apply = function(compiler) {
    compiler.plugin("compilation", function(compilation) {
      compilation.plugin("build-module", function(module) {
      });
      compilation.plugin("optimize-chunk-assets", function(chunks, callback) {
        // Uglify 逻辑
      });
      compilation.plugin("normal-module-loader", function(context) {
      });
    });
  };
  ```

  在 webpack 中你经常可以看到 compilation.plugin(‘xxx’, callback) ，你可以把它当作是一个事件的绑定，这些事件在打包时由 webpack 来触发。

## 流程总览

在具体流程学习前，可以先通过这幅 [webpack 整体流程图](https://img.alicdn.com/tps/TB1GVGFNXXXXXaTapXXXXXXXXXX-4436-4244.jpg) 了解一下大致流程（建议保存下来查看）。

![TB1GVGFNXXXXXaTapXXXXXXXXXX-4436-4244](https://tva1.sinaimg.cn/large/006y8mN6ly1g6j5wkkj2jj30vd0u04qp.jpg)

## shell 与 config 解析

每次在命令行输入 webpack 后，操作系统都会去调用 `./node_modules/.bin/webpack` 这个 shell 脚本。这个脚本会去调用 `./node_modules/webpack/bin/webpack.js` 并追加输入的参数，如 -p , -w 。(图中 webpack.js 是 webpack 的启动文件，而 $@ 是后缀参数)

![img](https://tva1.sinaimg.cn/large/006y8mN6ly1g6j5wp0xr3j30dw03374c.jpg)

在 webpack.js 这个文件中 webpack 通过 optimist 将用户配置的 webpack.config.js 和 shell 脚本传过来的参数整合成 options 对象传到了下一个流程的控制对象中。

### 1. optimist

和 commander 一样，[optimist](https://github.com/substack/node-optimist) 实现了 node 命令行的解析，其 API 调用非常方便。

```javascript
var optimist = require("optimist");

optimist
  .boolean("json").alias("json", "j").describe("json")
  .boolean("colors").alias("colors", "c").describe("colors")
  .boolean("watch").alias("watch", "w").describe("watch")
  ...
```

获取到后缀参数后，optimist 分析参数并以键值对的形式把参数对象保存在 optimist.argv 中，来看看 argv 究竟有什么？

```javascript
// webpack --hot -w
{
  hot: true,
  profile: false,
  watch: true,
  ...
}
```

### 2. config 合并与插件加载

在加载插件之前，webpack 将 webpack.config.js 中的各个配置项拷贝到 options 对象中，并加载用户配置在 webpack.config.js 的 plugins 。接着 optimist.argv 会被传入到 `./node_modules/webpack/bin/convert-argv.js` 中，通过判断 argv 中参数的值决定是否去加载对应插件。(至于 webpack 插件运行机制，在之后的运行机制篇会提到)

```javascript
ifBooleanArg("hot", function() {
  ensureArray(options, "plugins");
  var HotModuleReplacementPlugin = require("../lib/HotModuleReplacementPlugin");
  options.plugins.push(new HotModuleReplacementPlugin());
});
...
return options;
```

`options` 作为最后返回结果，包含了之后构建阶段所需的重要信息。

```javascript
{ 
  entry: {},//入口配置
  output: {}, //输出配置
  plugins: [], //插件集合(配置文件 + shell指令) 
  module: { loaders: [ [Object] ] }, //模块配置
  context: //工程路径
  ... 
}
```

这和 webpack.config.js 的配置非常相似，只是多了一些经 shell 传入的插件对象。插件对象一初始化完毕， options 也就传入到了下个流程中。

```javascript
var webpack = require("../lib/webpack.js");
var compiler = webpack(options);
```

## 编译与构建流程

在加载配置文件和 shell 后缀参数申明的插件，并传入构建信息 options 对象后，开始整个 webpack 打包最漫长的一步。而这个时候，真正的 webpack 对象才刚被初始化，具体的初始化逻辑在 `lib/webpack.js` 中，如下：

```javascript
function webpack(options) {
  var compiler = new Compiler();
  ...// 检查options,若watch字段为true,则开启watch线程
  return compiler;
}
...
```

webpack 的实际入口是 Compiler 中的 run 方法，run 一旦执行后，就开始了编译和构建流程 ，其中有几个比较关键的 webpack 事件节点。

- `compile` 开始编译
- `make` 从入口点分析模块及其依赖的模块，创建这些模块对象
- `build-module` 构建模块
- `after-compile` 完成构建
- `seal` 封装构建结果
- `emit` 把各个chunk输出到结果文件
- `after-emit` 完成输出

### 1. 核心对象 Compilation

compiler.run 后首先会触发 compile ，这一步会构建出 Compilation 对象：

![compilation类图](https://tva1.sinaimg.cn/large/006y8mN6ly1g6j5wtuu15j30j90q4ta1.jpg)

#### 这个对象有**两个作用**:

* 一是负责组织整个打包过程，包含了每个构建环节及输出环节所对应的方法，可以从图中看到比较关键的步骤，如 `addEntry()` , `_addModuleChain()` ,`buildModule()` , `seal()` , `createChunkAssets()` (在每一个节点都会触发 webpack 事件去调用各插件)。
* 二是该对象内部存放着所有 module ，chunk，生成的 asset 以及用来生成最后打包文件的 template 的信息。

### 2. 编译与构建主流程

在创建 module 之前，Compiler 会触发 make，并调用 `Compilation.addEntry` 方法，通过 options 对象的 entry 字段找到我们的入口js文件。之后，在 addEntry 中调用私有方法 `_addModuleChain` ，这个方法主要做了两件事情。一是根据模块的类型获取对应的模块工厂并创建模块，二是构建模块。

#### 而构建模块作为最耗时的一步，又可细化为三步：

- 调用各 loader 处理模块之间的依赖

  webpack 提供的一个很大的便利就是能将所有资源都整合成模块，不仅仅是 js 文件。所以需要一些 loader ，比如 `url-loader` ， `jsx-loader` ， `css-loader` 等等来让我们可以直接在源文件中引用各类资源。webpack 调用 `doBuild()` ，对每一个 require() 用对应的 loader 进行加工，最后生成一个 js module。

  ```javascript
  Compilation.prototype._addModuleChain = function process(context, dependency, onModule, callback) {
    var start = this.profile && +new Date();
    ...
    // 根据模块的类型获取对应的模块工厂并创建模块
    var moduleFactory = this.dependencyFactories.get(dependency.constructor);
    ...
    moduleFactory.create(context, dependency, function(err, module) {
      var result = this.addModule(module);
      ...
      this.buildModule(module, function(err) {
        ...
        // 构建模块，添加依赖模块
      }.bind(this));
    }.bind(this));
  };
  ```

- 调用 [acorn](https://github.com/ternjs/acorn) 解析经 loader 处理后的源文件生成抽象语法树 AST

  ```javascript
   Parser.prototype.parse = function parse(source, initialState) {
    var ast;
    if (!ast) {
      // acorn以es6的语法进行解析
      ast = acorn.parse(source, {
        ranges: true,
        locations: true,
        ecmaVersion: 6,
        sourceType: "module"
      });
    }
    ...
  };
  ```

- 遍历 AST，构建该模块所依赖的模块

  对于当前模块，或许存在着多个依赖模块。当前模块会开辟一个依赖模块的数组，在遍历 AST 时，将 require() 中的模块通过 `addDependency()` 添加到数组中。当前模块构建完成后，webpack 调用 `processModuleDependencies` 开始递归处理依赖的 module，接着就会重复之前的构建步骤。

  ```javascript
   Compilation.prototype.addModuleDependencies = function(module, dependencies, bail, cacheGroup, recursive, callback) {
    // 根据依赖数组(dependencies)创建依赖模块对象
    var factories = [];
    for (var i = 0; i < dependencies.length; i++) {
      var factory = _this.dependencyFactories.get(dependencies[i][0].constructor);
      factories[i] = [factory, dependencies[i]];
    }
    ...
    // 与当前模块构建步骤相同
  }
  ```

### 3. 构建细节

module 是 webpack 构建的核心实体，也是所有 module 的 父类，它有几种不同子类：`NormalModule` , `MultiModule` , `ContextModule` , `DelegatedModule` 等。但这些核心实体都是在构建中都会去调用对应方法，也就是 `build()` 。来看看其中具体做了什么：

```javascript
// 初始化module信息，如context,id,chunks,dependencies等。
NormalModule.prototype.build = function build(options, compilation, resolver, fs, callback) {
  this.buildTimestamp = new Date().getTime(); // 构建计时
  this.built = true;
  return this.doBuild(options, compilation, resolver, fs, function(err) {
    // 指定模块引用，不经acorn解析
    if (options.module && options.module.noParse) {
      if (Array.isArray(options.module.noParse)) {
        if (options.module.noParse.some(function(regExp) {
            return typeof regExp === "string" ?
            this.request.indexOf(regExp) === 0 :
              regExp.test(this.request);
          }, this)) {
          return callback();
        }
      } else if (typeof options.module.noParse === "string" ?
        this.request.indexOf(options.module.noParse) === 0 :
          options.module.noParse.test(this.request)) {
        return callback();
      }
    }
    // 由acorn解析生成ast
    try {
      this.parser.parse(this._source.source(), {
        current: this,
        module: this,
        compilation: compilation,
        options: options
      });
    } catch (e) {
      var source = this._source.source();
      this._source = null;
      return callback(new ModuleParseError(this, source, e));
    }
    return callback();
  }.bind(this));
};
```

对于每一个 module ，它都会有这样一个构建方法。当然，它还包括了从构建到输出的一系列的有关 module 生命周期的函数，我们通过 module 父类类图其子类类图(这里以 NormalModule 为例)来观察其真实形态：

![module类图](https://tva1.sinaimg.cn/large/006y8mN6ly1g6j5wzssoaj30cd0y4t9d.jpg)

可以看到无论是构建流程，处理依赖流程，包括后面的封装流程都是与 module 密切相关的。

## 打包输出

在所有模块及其依赖模块 build 完成后，webpack 会监听 `seal` 事件调用各插件对构建后的结果进行封装，要逐次对每个 module 和 chunk 进行整理，生成编译后的源码，合并，拆分，生成 hash 。 同时这是我们在开发时进行代码优化和功能添加的关键环节。

```javascript
Compilation.prototype.seal = function seal(callback) {
  this.applyPlugins("seal"); // 触发插件的seal事件
  this.preparedChunks.sort(function(a, b) {
    if (a.name < b.name) {
      return -1;
    }
    if (a.name > b.name) {
      return 1;
    }
    return 0;
  });
  this.preparedChunks.forEach(function(preparedChunk) {
    var module = preparedChunk.module;
    var chunk = this.addChunk(preparedChunk.name, module);
    chunk.initial = chunk.entry = true;
    // 整理每个Module和chunk，每个chunk对应一个输出文件。
    chunk.addModule(module);
    module.addChunk(chunk);
  }, this);
  this.applyPluginsAsync("optimize-tree", this.chunks, this.modules, function(err) {
    if (err) {
      return callback(err);
    }
    ... // 触发插件的事件
    this.createChunkAssets(); // 生成最终assets
    ... // 触发插件的事件
  }.bind(this));
};
```

#### 1. 生成最终 assets

在封装过程中，webpack 会调用 Compilation 中的 `createChunkAssets` 方法进行打包后代码的生成。 createChunkAssets 流程如下：

![createChunkAssets流程](https://tva1.sinaimg.cn/large/006y8mN6ly1g6j5x54h28j30qn0mf74x.jpg)

- 不同的 Template

  从上图可以看出通过判断是入口 js 还是需要异步加载的 js 来选择不同的模板对象进行封装，入口 js 会采用 webpack 事件流的 render 事件来触发 `Template类` 中的 `renderChunkModules()` (异步加载的 js 会调用 chunkTemplate 中的 render 方法)。

  ```javascript
  if(chunk.entry) {
    source = this.mainTemplate.render(this.hash, chunk, this.moduleTemplate, this.dependencyTemplates);
  } else {
    source = this.chunkTemplate.render(chunk, this.moduleTemplate, this.dependencyTemplates);
  }
  ```

  在 webpack 中有四个 Template 的子类，分别是 `MainTemplate.js` ， `ChunkTemplate.js` ，`ModuleTemplate.js` ， `HotUpdateChunkTemplate.js` ，前两者先前已大致有介绍，而 ModuleTemplate 是对所有模块进行一个代码生成，HotUpdateChunkTemplate 是对热替换模块的一个处理。

- 模块封装

  模块在封装的时候和它在构建时一样，都是调用各模块类中的方法。封装通过调用 `module.source()` 来进行各操作，比如说 require() 的替换。

  ```javascript
  MainTemplate.prototype.requireFn = "__webpack_require__";
  MainTemplate.prototype.render = function(hash, chunk, moduleTemplate, dependencyTemplates) {
      var buf = [];
      // 每一个module都有一个moduleId,在最后会替换。
      buf.push("function " + this.requireFn + "(moduleId) {");
      buf.push(this.indent(this.applyPluginsWaterfall("require", "", chunk, hash)));
      buf.push("}");
      buf.push("");
      ... // 其余封装操作
  };
  ```

- 生成 assets

  各模块进行 doBlock 后，把 module 的最终代码循环添加到 source 中。一个 source 对应着一个 asset 对象，该对象保存了单个文件的文件名( name )和最终代码( value )。

#### 2. 输出

最后一步，webpack 调用 Compiler 中的 `emitAssets()` ，按照 output 中的配置项将文件输出到了对应的 path 中，从而 webpack 整个打包过程结束。要注意的是，若想对结果进行处理，则需要在 `emit` 触发后对自定义插件进行扩展。

## 总结

webpack 的整体流程主要还是依赖于 `compilation` 和 `module` 这两个对象，但其思想远不止这么简单。最开始也说过，webpack 本质是个插件集合，并且由 `tapable` 控制各插件在 webpack 事件流上运行。同时，在业务开发中，无论是为了提升构建效率，或是减小打包文件大小，我们都可以通过编写 webpack 插件来进行流程上的控制，这个也会在之后提到

# Webpack 4 

## 特点：

Webpack 4 是最流行的模块捆绑器之一。它提供了很多新特性，例如零配置、合理的默认值、性能改进、开箱即用的优化工具，等等。

零配置：Webpack 4 不需要配置文件，这是 Webpack 4 的一个全新的变更。

性能提升：Webpack 4 是迄今为止最快的 Webpack 版本。

合理的默认值：Webpack 4 的主要概念包括入口、输出、加载器和插件。

## webpack4和之前版本里一些主要的变化：

1、webpack不再支持node v4，这是因为新的webpack和附属插件使用了es6的语法，v4版本对es6不是很到位，所以，就不伺候了。

2、开始采取约定优于配置的思路，webpack@4.x里把很多选项都设置了默认值，比如入口就是./src，输出目录就是./dist，当然如果你自己去设置，它也不会拦着。所以在用webpack@4.x的时候，我们甚至都可以没有webpack.config.js这个配置文件也能一样打包。

3、拆分了旧版本里的webpack，分成了用来处理逻辑的webpack和提供可执行命令的webpack-cli，这也是有的同学把webpack装成新版本以后会报找不到webpack-cli模块这个错误的原因。

4、添加了mode选项，用来区分编译的环境，提供了development、production和none三个选项，理论上这个选项是强制指定的，其实不指定的话它也是有默认值的，但会给出一个warning，官方这么强烈建议了，就显式的指定一下吧。指定方式有两种，一是在启动命令里：

```text
./node_modules/.bin/webpack --mode production
```

另外一种就是在配置文件里指定，方式如下：

```text
var config = {
    mode: 'production'  // 可选development、production和none
}
```

5、配置上的变动，类似删除了commonChunksPlugin，用optimization来代替这种。还有loader的用法也和1.15.0不一样了，但这个升级是在之前版本里就有的，不是webpack@4.x带来的。

6、性能优化，提高了打包性能。另外从webpack2起，引入了Tree-shaking机制来提出没有被引用的模块。它的原理是按着引用关系重新建立一个新的依赖树，而没有被引用的模块就不会被打包到最后的代码里。之前的压缩优化方式是先把所有模块都挂到树上，然后通过分析后，删掉没被引用的模块。从字面意思来看，我觉得原来的方式更像在摇树，把没用的摇下来。这两种方式最后压缩完的结果可能类似，但设计思路完全是从两个方向走的。

7、其他。以上这些就是一些比较主要的变化，更详细的升级文档可以参考下官方的说明：[https://github.com/webpack/webpack/releases]