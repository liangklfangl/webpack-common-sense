#### 1.获取webpack配置的输出配置

```js
 compilation.outputOptions
```

其中该对象有如下内容:

```js
 { path: 'builds',
     filename: 'bundle.js',
     publicPath: 'builds/',
     chunkFilename: '[id].bundle.js',
     library: '',
     hotUpdateFunction: 'webpackHotUpdate',
     jsonpFunction: 'webpackJsonp',
     libraryTarget: 'var',
     sourceMapFilename: '[file].map[query]',
     hotUpdateChunkFilename: '[id].[hash].hot-update.js',
     hotUpdateMainFilename: '[hash].hot-update.json',
     crossOriginLoading: false,
     hashFunction: 'md5',
     hashDigest: 'hex',
     hashDigestLength: 20,
     devtoolLineToLine: false,
     strictModuleExceptionHandling: false }
```

如果你需要更多的信息只能通过如下：

```js
compilation.options
```
<pre>
  {
   entry: './src',
  output:
   { path: 'builds',
     filename: 'bundle.js',
     publicPath: 'builds/',
     chunkFilename: '[id].bundle.js',
     library: '',
     hotUpdateFunction: 'webpackHotUpdate',
     jsonpFunction: 'webpackJsonp',
     libraryTarget: 'var',
     sourceMapFilename: '[file].map[query]',
     hotUpdateChunkFilename: '[id].[hash].hot-update.js',
     hotUpdateMainFilename: '[hash].hot-update.json',
     crossOriginLoading: false,
     hashFunction: 'md5',
     hashDigest: 'hex',
     hashDigestLength: 20,
     devtoolLineToLine: false,
     strictModuleExceptionHandling: false },
  plugins:
   [ CommonsChunkPlugin {
       chunkNames: 'vendor',
       filenameTemplate: 'vendor.bundle.js',
       minChunks: 2,
       selectedChunks: undefined,
       async: undefined,
       minSize: undefined,
       ident: 'C:\\Users\\Administrator\\Desktop\\webpack-chunkfilename\\node_mo
dules\\webpack\\lib\\optimize\\CommonsChunkPlugin.js0' },
     HtmlWebpackPlugin {
       options: [Object],
       childCompilerHash: '729d3caf962246f308dc6d5b1542a9ae',
       childCompilationOutputName: 'index.html' } ],
  module:
   { loaders: [ [Object], [Object], [Object], [Object] ],
     unknownContextRequest: '.',
     unknownContextRegExp: false,
     unknownContextRecursive: true,
     unknownContextCritical: true,
     exprContextRequest: '.',
     exprContextRegExp: false,
     exprContextRecursive: true,
     exprContextCritical: true,
     wrappedContextRegExp: /.*/,
     wrappedContextRecursive: true,
     wrappedContextCritical: false,
     unsafeCache: true },
  bail: false,
  profile: false,
  context: 'C:\\Users\\Administrator\\Desktop\\webpack-chunkfilename',
  devtool: false,
  cache: true,
  target: 'web',
  <!--webpack中加入了node模块-->
  node:
   { console: false,
     process: true,
     global: true,
     Buffer: true,
     setImmediate: true,
     __filename: 'mock',
     __dirname: 'mock' },
  performance: { maxAssetSize: 250000, maxEntrypointSize: 250000, hints: false }
,
  resolve:
   { unsafeCache: true,
     modules: [ 'node_modules' ],
     extensions: [ '.js', '.json' ],
     aliasFields: [ 'browser' ],
     mainFields: [ 'browser', 'module', 'main' ] },
  resolveLoader:
   { unsafeCache: true,
     mainFields: [ 'loader', 'main' ],
     extensions: [ '.js', '.json' ] } }
</pre>

注意我们其中关注的几点，node模块!

下面给出一个获取publicPath(输出配置)路径的例子:

```js
 var webpackStatsJson = compilation.getStats().toJson();
  //获取compilation的所有的信息
  // Use the configured public path or build a relative path
  var publicPath = typeof compilation.options.output.publicPath !== 'undefined'
    // If a hard coded public path exists use it
    ? compilation.mainTemplate.getPublicPath({hash: webpackStatsJson.hash})
    // If no public path was set get a relative url path
    // 这个publicPath是可以使用hash的
    : path.relative(path.resolve(compilation.options.output.path, path.dirname(self.childCompilationOutputName)), compilation.options.output.path)
      .split(path.sep).join('/');
    //self.childCompilationOutputName得到的"index.html"，dirname得到的是".",所以resolve结果为" C:\Users\Administrator\Desktop\webpack-chunkfilename\builds"
    if (publicPath.length && publicPath.substr(-1, 1) !== '/') {
      publicPath += '/';
    }
    //获取倒数第一个字符,添加一个"/"
```

很显然我们的publicPath是可以配置`hash参数`的，如果没有配置publicPath那么我们是相对于output.path目录来说的！


#### 2.如何获取启动一个子compiler中的错误信息

```js
if (childCompilation && childCompilation.errors && childCompilation.errors.length) {
        var errorDetails = childCompilation.errors.map(function (error) {
          return error.message + (error.error ? ':\n' + error.error : '');
        }).join('\n');
        reject(new Error('Child compilation failed:\n' + errorDetails));
        //如果报错，直接reject
      }
```

其实对于compiler对象，而不是childCompiler对象也是同样的道理


#### 3.开启的child compiler如何对文件中的hash等进行替换得到最终文件名

```js
   //'assets-path'钩子函数来完成文件名替
   var outputName = compilation.mainTemplate.applyPluginsWaterfall('asset-path', outputOptions.filename, {
          hash: childCompilation.hash,
          chunk: entries[0]
          //因为上面提供的是SingleEntryPlugin ·
        });
```

注意：此处调用的是compilation而不是childCompilation完成的!

#### 4.如何加载本地文件系统的文件然后开启一个新的child c compiler

```js
HtmlWebpackPlugin.prototype.getFullTemplatePath = function (template, context) {
  // If the template doesn't use a loader use the lodash template loader
  if (template.indexOf('!') === -1) {
    template = require.resolve('./lib/loader.js') + '!' + path.resolve(context, template);
    //对于我们的这个ejs文件，我们使用特定的loader加载
  }
  // Resolve template path
  return template.replace(
    /([!])([^/\\][^!?]+|[^/\\!?])($|\?[^!?\n]+$)/,
    function (match, prefix, filepath, postfix) {
      return prefix + path.resolve(filepath) + postfix;
    });
};
//通过上面的getFullTemplatePath得到通过loader加载文件后的完整的形式，最后传入到我们的SingleEntryPlugin中
  childCompiler.apply(
    new NodeTemplatePlugin(outputOptions),
    new NodeTargetPlugin(),
    new LibraryTemplatePlugin('HTML_WEBPACK_PLUGIN_RESULT', 'var'),
    new SingleEntryPlugin(this.context, template),
    //上面的通过loader加载的本地文件系统的内容传入到我们的SingleEntryPlugin中
    new LoaderTargetPlugin('node')
  );
```

所以上面的传入到中的内容大致如下(其中!后面表示要加载的文件，前面表示相应的loader):

C:\Users\Administrator\Desktop\webpack-chunkfilename\node_modules\html-webpack-plugin\lib\loader.js!C:\Users\Administrator\Desktop\w
 ebpack-chunkfilename\node_modules\html-webpack-plugin\default_index.ejs

#### 4.获取stats中所有的chunks

 获取compilation所有的chunks:

```js
compilation.getStats().toJson().chunks
```

chunks所有的内容如下：

<pre>
[ { id: 0,//chunk id
    rendered: true,
    initial: false,//require.ensure产生，非initial
    entry: false,//非入口文件
    recorded: undefined,
    extraAsync: false,
    size: 296855,//chunk大小，比特
    names: [],//require.ensure不是通过webpack配置的，所以chunk的names是空
    files: [ '0.bundle.js' ],//该chunk产生的文件
    hash: '42fbfbea594ba593e76a',//chunk的hash
    parents: [ 2 ],//父级chunk
    origins: [ [Object] ] },
  { id: 1,
    rendered: true,
    initial: false,//require.ensure产生，非initial
    entry: false,//非入口文件
    recorded: undefined,
    extraAsync: false,
    size: 297181,//chunk大小，比特
    names: [],
    files: [ '1.bundle.js' ],//产生的文件
    hash: '456d05301e4adca16986',//chunk的hash
    parents: [ 2 ],
    origins: [ [Object] ] },
  { id: 2,
    rendered: true,
    initial: true,//commonchunkplugin产生或者入口文件产生
    entry: false,//非入口文件
    recorded: undefined,
    extraAsync: false,
    size: 687,//chunk大小，比特
    names: [ 'main' ],
    files: [ 'bundle.js' ],//产生的文件
    hash: '248029a0cfd99f46babc',//chunk的hash
    parents: [ 3 ],
    origins: [ [Object] ] },
  { id: 3,
    rendered: true,
    initial: true,//monchunkplugin产生或者入口文件产生
    entry: true,//commonchunkplugin把webpack执行环境抽取出来
    recorded: undefined,
    extraAsync: false,
    size: 0,//chunk大小，比特
    names: [ 'vendor' ],
    files: [ 'vendor.bundle.js' ],//产生的文件
    hash: 'fbf76c7c330eaf0de943',//chunk的hash
    parents: [],
    origins: [] } ]
</pre>

注意chunk的names是一个数组！

下面是给出的几个关于chunks的例子：

例1：html-webpack-plugin中就使用到了多个chunks属性,如names,initial等

```js
//该chunk要被选中的条件是：有名称，不是懒加载，在includedChunks中但是不在excludedChunks中
HtmlWebpackPlugin.prototype.filterChunks = function (chunks, includedChunks, excludedChunks) {
  return chunks.filter(function (chunk) {
    var chunkName = chunk.names[0];
    // This chunk doesn't have a name. This script can't handled it.
    //通过require.ensure产生的chunk不会被保留，names是一个数组
    if (chunkName === undefined) {
      return false;
    }
    // Skip if the chunk should be lazy loaded
    //如果是require.ensure产生的chunk直接忽略
    if (!chunk.initial) {
      return false;
    }
    // Skip if the chunks should be filtered and the given chunk was not added explicity
    //这个chunk必须在includedchunks里面
    if (Array.isArray(includedChunks) && includedChunks.indexOf(chunkName) === -1) {
      return false;
    }
    // Skip if the chunks should be filtered and the given chunk was excluded explicity
    //这个chunk不能在excludedChunks中
    if (Array.isArray(excludedChunks) && excludedChunks.indexOf(chunkName) !== -1) {
      return false;
    }
    // Add otherwise
    return true;
  });
};
```

例2：通过id对chunks进行排序

```js
/**
 * Sorts the chunks based on the chunk id.
 *
 * @param  {Array} chunks the list of chunks to sort
 * @return {Array} The sorted list of chunks
 * entry chunk在前，两个都是entry那么id大的在前
 */
module.exports.id = function (chunks) {
  return chunks.sort(function orderEntryLast (a, b) {
    if (a.entry !== b.entry) {
      return b.entry ? 1 : -1;
    } else {
      return b.id - a.id;
    }
  });
};
```

这样入口文件都会排列在前面，但是有一点就是这里的id较大的在前面，这和我们的occurenceOrderplugin完全是相反的，所以这里还是存在疑问的。

例3:通过chunk.parents(全部是parentId数组)来获取拓排序

```js
/*
  Sorts dependencies between chunks by their "parents" attribute.
  This function sorts chunks based on their dependencies with each other.
  The parent relation between chunks as generated by Webpack for each chunk
  is used to define a directed (and hopefully acyclic) graph, which is then
  topologically sorted in order to retrieve the correct order in which
  chunks need to be embedded into HTML. A directed edge in this graph is
  describing a "is parent of" relationship from a chunk to another (distinct)
  chunk. Thus topological sorting orders chunks from bottom-layer chunks to
  highest level chunks that use the lower-level chunks.

  @param {Array} chunks an array of chunks as generated by the html-webpack-plugin.
  It is assumed that each entry contains at least the properties "id"
  (containing the chunk id) and "parents" (array containing the ids of the
  parent chunks).
  @return {Array} A topologically sorted version of the input chunks
  因为最上面的通过commonchunkplugin产生的chunk具有webpack的runtime，所以排列在前面
*/
module.exports.dependency = function (chunks) {
  if (!chunks) {
    return chunks;
  }
  // We build a map (chunk-id -> chunk) for faster access during graph building.
  // 通过chunk-id -> chunk这种Map结构更加容易绘制图
  var nodeMap = {};
  chunks.forEach(function (chunk) {
    nodeMap[chunk.id] = chunk;
  });
  // Next, we add an edge for each parent relationship into the graph
  var edges = [];
  chunks.forEach(function (chunk) {
    if (chunk.parents) {
      // Add an edge for each parent (parent -> child)
      chunk.parents.forEach(function (parentId) {
        // webpack2 chunk.parents are chunks instead of string id(s)
        var parentChunk = _.isObject(parentId) ? parentId : nodeMap[parentId];
        // If the parent chunk does not exist (e.g. because of an excluded chunk)
        // we ignore that parent
        if (parentChunk) {
          edges.push([parentChunk, chunk]);
        }
      });
    }
  });
  // We now perform a topological sorting on the input chunks and built edges
  return toposort.array(chunks, edges);
};

```

通过这种方式可以把各个chunk公有的模块排列在前面，从而提前加载，这是合理的！

#### 5.获取stats中所有的assets
 获取compilation所有的assets:

```js
compilation.getStats().toJson().assets
```

assets内部结构如下：

<pre>
 [ { name: '0.bundle.js',
    size: 299109,
    chunks: [ 0, 3 ],
    <!-- 公共模块被抽取到vendor.bundle.js -->
    chunkNames: [],
    emitted: undefined,
    isOverSizeLimit: undefined },
  { name: '1.bundle.js',
    size: 299469,
    chunks: [ 1, 3 ],
    chunkNames: [],
    emitted: undefined,
    isOverSizeLimit: undefined },
  { name: 'bundle.js',
    <!-- 产生的文件名称 -->
    size: 968,
    <!-- 文件大小 -->
    chunks: [ 2, 3 ],
    <!-- 这个输出资源对应的chunk(有一部分被提取到vendor.js,如runtime) -->
    chunkNames: [ 'main' ],
    <!-- 这个输出资源对应的chunk的名称 -->
    emitted: undefined,
    <!-- 是否又重新生成了该emitted，用于判断资源有没有变化 -->
    isOverSizeLimit: undefined },
  { name: 'vendor.bundle.js',
    size: 5562,
    chunks: [ 3 ],
    chunkNames: [ 'vendor' ],
    emitted: undefined,
    isOverSizeLimit: undefined }]
</pre>

上次看到webpack-dev-server源码的时候看到了如何判断该assets是否变化的判断，其就是通过上面的emitted来判断的:

```js
compiler.plugin("done", function(stats) {
    this._sendStats(this.sockets, stats.toJson(clientStats));
    //clientStats表示客户端stats要输出的内容过滤
    this._stats = stats;
  }.bind(this));
Server.prototype._sendStats = function(sockets, stats, force) {
  if(!force &&
    stats &&
    (!stats.errors || stats.errors.length === 0) &&
    stats.assets &&
    stats.assets.every(function(asset) {
      return !asset.emitted;
      //每一个asset都是没有emitted属性，表示没有发生变化。如果发生变化那么这个assets肯定有emitted属性。所以emitted属性表示是否又重新生成了一遍assets资源
    })
  )
    return this.sockWrite(sockets, "still-ok");
  this.sockWrite(sockets, "hash", stats.hash);
  //正常情况下首先发送hash，然后发送ok
  if(stats.errors.length > 0)
    this.sockWrite(sockets, "errors", stats.errors);
  else if(stats.warnings.length > 0)
    this.sockWrite(sockets, "warnings", stats.warnings);
  else
  //发送hash后再发送ok
    this.sockWrite(sockets, "ok");
}
```


#### 6.获取stats中所有的modules

 获取compilation所有的modules:

```js
compilation.getStats().toJson().modules
```

modules内部结构如下：

```js
 { id: 10,
   identifier: 'C:\\Users\\Administrator\\Desktop\\webpack-chunkfilename\\node_
odules\\html-loader\\index.js!C:\\Users\\Administrator\\Desktop\\webpack-chunkf
lename\\src\\Components\\Header.html',
   name: './src/Components/Header.html',//模块名称，已经转化为相对于根目录的路径
   index: 10,
   index2: 8,
   size: 62,
   cacheable: true,//缓存
   built: true,
   optional: false,
   prefetched: false,
   chunks: [ 0 ],//在那个chunk中出现
   assets: [],
   issuer: 'C:\\Users\\Administrator\\Desktop\\webpack-chunkfilename\\node_modu
es\\eslint-loader\\index.js!C:\\Users\\Administrator\\Desktop\\webpack-chunkfil
name\\src\\Components\\Header.js',//是谁开始本模块的调用的，即模块调用发起者
   issuerId: 1,//发起者id
   issuerName: './src/Components/Header.js',//发起者相对于根目录的路径
   profile: undefined,
   failed: false,
   errors: 0,
   warnings: 0,
   reasons: [ [Object] ],
   usedExports: [ 'default' ],
   providedExports: null,
   depth: 2,
   source: 'module.exports = "<header class=\\"header\\">{{text}}</header>";' }//source是模块内容
```

#### 7.如何对输出的assets资源进行过滤

下面给出一个例子1:

```js
 var assets = {
    // The public path
    publicPath: publicPath,
    // Will contain all js & css files by chunk
    chunks: {},
    // Will contain all js files
    js: [],
    // Will contain all css files
    css: [],
    // Will contain the html5 appcache manifest files if it exists
    //这里是application cache文件，这里不是文件内容是文件的名称。key就是文件名称
    manifest: Object.keys(compilation.assets).filter(function (assetFile) {
      return path.extname(assetFile) === '.appcache';
    })[0]
  };
```

例子2：过滤出所有的css文件

```js
 var chunkFiles = [].concat(chunk.files).map(function (chunkFile) {
      return publicPath + chunkFile;
    });
    var css = chunkFiles.filter(function (chunkFile) {
      // Some chunks may contain content hash in their names, for ex. 'main.css?1e7cac4e4d8b52fd5ccd2541146ef03f'.
      // We must proper handle such cases, so we use regexp testing here
      return /.css($|\?)/.test(chunkFile);
    });
```

当然过滤出来的资源可能会要添加hash，我们看看如何处理：

```js
/**
 * Appends a cache busting hash
 self.appendHash(assets.manifest, webpackStatsJson.hash);
 为文件名称后面添加一个hash值用于缓存，是在文件的路径上而不是内容
 */
 var webpackStatsJson = compilation.getStats().toJson();
 if (this.options.hash) {
    assets.manifest = self.appendHash(assets.manifest, webpackStatsJson.hash);
    assets.favicon = self.appendHash(assets.favicon, webpackStatsJson.hash);
  }
HtmlWebpackPlugin.prototype.appendHash = function (url, hash) {
  if (!url) {
    return url;
  }
  return url + (url.indexOf('?') === -1 ? '?' : '&') + hash;
};
```

#### 8.如何对输出的资源路径进行特别的处理

```js
HtmlWebpackPlugin.prototype.htmlWebpackPluginAssets = function (compilation, chunks) {
  var self = this;
  var webpackStatsJson = compilation.getStats().toJson();
  //获取compilation的所有的信息
  // Use the configured public path or build a relative path
  var publicPath = typeof compilation.options.output.publicPath !== 'undefined'
    // If a hard coded public path exists use it
    ? compilation.mainTemplate.getPublicPath({hash: webpackStatsJson.hash})
    // If no public path was set get a relative url path
    // 这个publicPath是可以使用hash的
    : path.relative(path.resolve(compilation.options.output.path, path.dirname(self.childCompilationOutputName)), compilation.options.output.path)
      .split(path.sep).join('/');
    //self.childCompilationOutputName得到的"index.html"，dirname得到的是".",所以resolve结果为" C:\Users\Administrator\Desktop\webpack-chunkfilename\builds"
    if (publicPath.length && publicPath.substr(-1, 1) !== '/') {
      publicPath += '/';
    }
    //获取倒数第一个字符,添加一个"/"

  var assets = {
    // The public path
    publicPath: publicPath,
    // Will contain all js & css files by chunk
    chunks: {},
    // Will contain all js files
    js: [],
    // Will contain all css files
    css: [],
    // Will contain the html5 appcache manifest files if it exists
    //这里是application cache文件，这里不是文件内容是文件的名称
    manifest: Object.keys(compilation.assets).filter(function (assetFile) {
      return path.extname(assetFile) === '.appcache';
    })[0]
  };

  // Append a hash for cache busting（缓存清除）
  //hash: true | false if true then append a unique webpack compilation hash to all
  // included scripts and CSS files. This is useful for cache busting.
  if (this.options.hash) {
    assets.manifest = self.appendHash(assets.manifest, webpackStatsJson.hash);
    assets.favicon = self.appendHash(assets.favicon, webpackStatsJson.hash);
  }
  for (var i = 0; i < chunks.length; i++) {
    var chunk = chunks[i];
    var chunkName = chunk.names[0];
    //为每一个chunk都在上面的这个assets对象上添加一个对象，如assets.chunks[chunkName]={}
    assets.chunks[chunkName] = {};
    // Prepend the public path to all chunk files
    //chunk.files表示该chunk产生的所有的文件，不过是文件名称name而不是内容
    var chunkFiles = [].concat(chunk.files).map(function (chunkFile) {
      return publicPath + chunkFile;
    });

    // Append a hash for cache busting
    //为每一个文件加上了publicPath同时还要加上hash
    if (this.options.hash) {
      chunkFiles = chunkFiles.map(function (chunkFile) {
        return self.appendHash(chunkFile, webpackStatsJson.hash);
      });
    }

    // Webpack outputs an array for each chunk when using sourcemaps
    // But we need only the entry file
    //chunk.files[0]就是该chunk产生的入口文件
    var entry = chunkFiles[0];
    assets.chunks[chunkName].size = chunk.size;
    assets.chunks[chunkName].entry = entry;
    assets.chunks[chunkName].hash = chunk.hash;
    assets.js.push(entry);
    //为每一个该chunk产生的文件都在上面的assets对象上添加一个对象，key是chunkName
    //value为一个对象{chunkName:{size:100,entry:'/qlin/',hash:'chunk的hash'}}
    // Gather all css files
    var css = chunkFiles.filter(function (chunkFile) {
      // Some chunks may contain content hash in their names, for ex. 'main.css?1e7cac4e4d8b52fd5ccd2541146ef03f'.
      // We must proper handle such cases, so we use regexp testing here
      return /.css($|\?)/.test(chunkFile);
    });
    assets.chunks[chunkName].css = css;
    //css属性就是我们的文件路径
    assets.css = assets.css.concat(css);
  }
  // Duplicate css assets can occur on occasion if more than one chunk
  // requires the same css.
  assets.css = _.uniq(assets.css);
  //如果多个chunk使用了同一个css那么会产生重复的css
  return assets;
};
```

上面的这个例子展示了如何如何为输出的资源做这些处理：添加publicPath以便html能够正常访问;为资源的文件名，即URL中添加编译的hash值。同时你要注意：

```js
  var entry = chunkFiles[0];
    assets.chunks[chunkName].size = chunk.size;
    assets.chunks[chunkName].entry = entry;
    assets.chunks[chunkName].hash = chunk.hash;
    assets.js.push(entry);
```

之所以会如此处理是因为如果在webpack中配置了：

```js
  devtool:'cheap-source-map',
```

那么每一个chunk产生的files就是一个数组，如下：

```js
[ 'builds/vendor.bundle.js', 'builds/vendor.bundle.js.map' ]
[ 'builds/bundle.js', 'builds/bundle.js.map' ]
```

所以我们只会保存第一个js文件而已！

#### 9.如何判断是否要重新产生资源文件

```js
 // If this is a hot update compilation, move on!
 // This solves a problem where an `index.html` file is generated for hot-update js files
 // It only happens in Webpack 2, where hot updates are emitted separately before the full bundle
  if (self.isHotUpdateCompilation(assets)) {
      return callback();
    }  // If the template and the assets did not change we don't have to emit the html
    //如果template和assets资源没有发生变化，我们不会重新产生html
    var assetJson = JSON.stringify(self.getAssetFiles(assets));
    //注意：这里包括了js,css,.appcache文件以及publicPath路径
    //isCompilationCached = compilationResult.hash && self.childCompilerHash === compilationResult.hash;
   //如果上次child compiler产生的hash和本次产生的hash(本次再次编译了一遍html)一致表示html没有变化
   //同时配置指定了可以读取html缓存，而且资源本身也没有发生变化，直接返回
    if (isCompilationCached && self.options.cache && assetJson === self.assetJson) {
      return callback();
    } else {
      self.assetJson = assetJson;
    }

HtmlWebpackPlugin.prototype.isHotUpdateCompilation = function (assets) {
  //如果每一个js的文件名，也就是入口文件都含有hot-update字段那么返回true
  return assets.js.length && assets.js.every(function (name) {
    return /\.hot-update\.js$/.test(name);
  });
};
HtmlWebpackPlugin.prototype.getAssetFiles = function (assets) {
  var files = _.uniq(Object.keys(assets).filter(function (assetType) {
    return assetType !== 'chunks' && assets[assetType];
    //获取类型不是chunks的资源
  }).reduce(function (files, assetType) {
    return files.concat(assets[assetType]);
  }, []));
  files.sort();
  return files;
};
```

如果该html是为了达到热更新的效果，那么不用每次都重新产生html文件，这有点值得学习。同时，这里也展示了如何判断本次编译的资源和上次的资源有没有发生变化，其通过把JSON转化为字符串的方式来比较！注意：childCompilerHash表示加载了html后得到的hash值！

#### 10.如何获取一次打包依赖的所有的文件并在所有文件中添加一个自己的文件

```js
/*
 * Pushes the content of the given filename to the compilation assets
 */
HtmlWebpackPlugin.prototype.addFileToAssets = function (filename, compilation) {
  filename = path.resolve(compilation.compiler.context, filename);
  //获取文件的绝对路径，我们配置的是一个path
  //处理一个 promise 的 map 集合。只有有一个失败，所有的执行都结束,也就是说我们必须要同时获取到这个文件的大小和内容本身
  return Promise.props({
    size: fs.statAsync(filename),
    source: fs.readFileAsync(filename)
  })
  .catch(function () {
    return Promise.reject(new Error('HtmlWebpackPlugin: could not load file ' + filename));
  })
  .then(function (results) {
    var basename = path.basename(filename);
    //path.basename('/foo/bar/baz/asdf/quux.html')
     // Returns: 'quux.html'
     //获取文件名称
    compilation.fileDependencies.push(filename);
    //在compilation.fileDependencies中添加文件，filename是完整的文件路径
    compilation.assets[basename] = {
      source: function () {
        return results.source;
      },
      //source是文件的内容，通过fs.readFileAsync完成
      size: function () {
        return results.size.size;
        //size通过 fs.statAsync(filename)完成
      }
    };
    return basename;
  });
};
```

这里要学习的内容还是挺多的，首先把我们自己的文件添加到需要打包依赖文件集合中：

```js
compilation.fileDependencies//在compilation.fileDependencies添加的必须是绝对路径
```

接下来就是修改compilation.assets对象：

```js
compilation.assets[basename] = {
      source: function () {
        return results.source;
      },
      //source是文件的内容，通过fs.readFileAsync完成
      size: function () {
        return results.size.size;
        //size通过 fs.statAsync(filename)完成
      }
    };
    return basename;
  });
```

不过这里的key就是我们的文件名，而不再包含文件路径！最后一个就是题外话了，来源于我们的[bluebird](http://bluebirdjs.com/docs/api-reference.html)(Promise.prop方法):

```js
var Promise = require('bluebird');
Promise.promisifyAll(fs);//所有fs对象的方法都promisify了，所以才有fs.statAsync(filename)， fs.readFileAsync(filename)，通过这两个方法就可以获取到我们的compilation.assets需要的source和size
```



参考资料：

[webpack-dev-middleware源码分析](https://github.com/liangklfang/webpack-dev-middleware)