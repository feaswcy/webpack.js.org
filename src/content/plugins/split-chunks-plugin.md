---
title: SplitChunksPlugin
contributors:
  - sokra
  - jeremenichelli
  - feaswcy
related:
  - title: "webpack 4: Code Splitting, chunk graph and the splitChunks optimization"
    url: https://medium.com/webpack/webpack-4-code-splitting-chunk-graph-and-the-splitchunks-optimization-be739a861366
---

一般来说，chunks 在webpack内部模块关系图中通过父子关系连接，`CommonsChunkPlugin` 这个插件用来避免在他们之间产生重复的依赖，但是进一步的优化是不太可能的。
从 webpack 4开始，`CommonsChunkPlugin` 被移除，取而代之的是配置选项中的`optimization.splitChunks` 和 `optimization.runtimeChunk` ，下面介绍新的配置是如何生效的。


## Defaults

Out of the box `SplitChunksPlugin` should work great for most users.
开箱即用的`SplitChunksPlugin`应该适合大多数用户

By default it only affects on-demand chunks because changing initial chunks would affect the script tags the HTML file should include to run the project.
默认情况下，它仅影响按需块，因为更改初始块会影响到HTML文件依赖的script文件以运行项目

在下面的情况下，webpack将会自动将chunk进行分割：
* chunk可以共享或者模块来自于`node_modules`文件夹
* 在min+gz之前，新的chunk将会超过30kb
* 模块按需加载时的最大并行请求数小于或等于5
* 初始页面加载时的最大并行请求数小于或等于3

webpack will automatically split chunks based on these conditions:

* New chunk can be shared OR modules are from the `node_modules` folder
* New chunk would be bigger than 30kb (before min+gz)
* Maximum number of parallel requests when loading chunks on demand would be lower or equal to 5
* Maximum number of parallel requests at initial page load would be lower or equal to 3

当试图满足最后两个条件时，会优先选择更大的块。 
When trying to fulfill the last two conditions, bigger chunks are preferred.

下面我们看一下例子
Let's take a look at some examples.

### Defaults: Example 1

``` js
// index.js

// dynamically import a.js
import("./a");
```

``` js
// a.js
import "react";

// ...
```

**Result:** A separate chunk would be created containing `react`. At the import call this chunk is loaded in parallel to the original chunk containing `./a`.

Why:

* Condition 1: The chunk contains modules from `node_modules`
* Condition 2: `react` is bigger than 30kb
* Condition 3: Number of parallel requests at the import call is 2
* Condition 4: Doesn't affect request at initial page load

What's the reasoning behind this? `react` probably won't change as often as your application code. By moving it into a separate chunk this chunk can be cached separately from your app code (assuming you are using chunkhash, records, Cache-Control or other long term cache approach).

### Defaults: Example 2

``` js
// entry.js

// dynamically import a.js and b.js
import("./a");
import("./b");
```

``` js
// a.js
import "./helpers"; // helpers is 40kb in size

// ...
```

``` js
// b.js
import "./helpers";
import "./more-helpers"; // more-helpers is also 40kb in size

// ...
```

**Result:** A separate chunk would be created containing `./helpers` and all dependencies of it. At the import calls this chunk is loaded in parallel to the original chunks.

Why:

* Condition 1: The chunk is shared between both import calls
* Condition 2: `helpers` is bigger than 30kb
* Condition 3: Number of parallel requests at the import calls is 2
* Condition 4: Doesn't affect request at initial page load

Putting the content of `helpers` into each chunk will result into its code being downloaded twice. By using a separate chunk this will only happen once. We pay the cost of an additional request, which could be considered a tradeoff. That's why there is a minimum size of 30kb.


## Configuration

For developers that want to have more control over this functionality, webpack provides a set of options to better fit your needs.

If you are manually changing the split configuration, measure the impact of the changes to see and make sure there's a real benefit.

W> Default configuration was chosen to fit web performance best practices but the optimum strategy for your project might defer depending on the nature of it.

### Configuring cache groups

The defaults assign all modules from `node_modules` to a cache group called `vendors` and all modules duplicated in at least 2 chunks to a cache group `default`.

A module can be assigned to multiple cache groups. The optimization then prefers the cache group with the higher `priority` (`priority` option) or that one that forms bigger chunks.

### Conditions

Modules from the same chunks and cache group will form a new chunk when all conditions are fulfilled.

There are 4 options to configure the conditions:

* `minSize` (default: 30000) Minimum size for a chunk.
* `minChunks` (default: 1) Minimum number of chunks that share a module before splitting
* `maxInitialRequests` (default 3) Maximum number of parallel requests at an entrypoint
* `maxAsyncRequests` (default 5) Maximum number of parallel requests at on-demand loading

### Naming

To control the chunk name of the split chunk the `name` option can be used.

W> When assigning equal names to different split chunks, all vendor modules are placed into a single shared chunk, though it's not recommend since it can result in more code downloaded.

The magic value `true` automatically chooses a name based on chunks and cache group key, otherwise a string or function can be passed.

When the name matches an entry point name, the entry point is removed.

#### `optimization.splitChunks.automaticNameDelimiter`

By default webpack will generate names using origin and name of the chunk, like `vendors~main.js`.

If your project has a conflict with the `~` character, it can be changed by setting this option to any other value that works for your project: `automaticNameDelimiter: "-"`.

Then the resulting names will look like `vendors-main.js`.

### Select modules

The `test` option controls which modules are selected by this cache group. Omitting it selects all modules. It can be a RegExp, string or function.

It can match the absolute module resource path or chunk names. When a chunk name is matched, all modules in this chunk are selected.

### Select chunks

With the `chunks` option the selected chunks can be configured.

There are 3 values possible `"initial"`, `"async"` and `"all"`. When configured the optimization only selects initial chunks, on-demand chunks or all chunks.

The option `reuseExistingChunk` allows to reuse existing chunks instead of creating a new one when modules match exactly.

This can be controlled per cache group.


### `optimization.splitChunks.chunks: all`

As it was mentioned before this plugin will affect dynamic imported modules. Setting the `optimization.splitChunks.chunks` option to `"all"` initial chunks will get affected by it (even the ones not imported dynamically). This way chunks can even be shared between entry points and on-demand loading.

This is the recommended configuration.

T> You can combine this configuration with the [HtmlWebpackPlugin](/plugins/html-webpack-plugin/), it will inject all the generated vendor chunks for you.


## `optimization.splitChunks`

This configuration object represents the default behavior of the `SplitChunksPlugin`.

```js
splitChunks: {
	chunks: "async",
	minSize: 30000,
	minChunks: 1,
	maxAsyncRequests: 5,
	maxInitialRequests: 3,
	automaticNameDelimiter: '~',
	name: true,
	cacheGroups: {
		vendors: {
			test: /[\\/]node_modules[\\/]/,
			priority: -10
		},
    default: {
			minChunks: 2,
			priority: -20,
			reuseExistingChunk: true
		}
	}
}
```

By default cache groups inherit options from `splitChunks.*`, but `test`, `priority` and `reuseExistingChunk` can only be configured on cache group level.

`cacheGroups` is an object where keys are the cache group names. All options from the ones listed above are possible: `chunks`, `minSize`, `minChunks`, `maxAsyncRequests`, `maxInitialRequests`, `name`.

You can set `optimization.splitChunks.cacheGroups.default` to `false` to disable the default cache group, same for `vendors` cache group.

The priority of the default groups are negative to allow any custom cache group to take higher priority (the default value is `0`).

Here are some examples and their effect:

### Split Chunks: Example 1

Create a `commons` chunk, which includes all code shared between entry points.

```js
splitChunks: {
	cacheGroups: {
		commons: {
			name: "commons",
			chunks: "initial",
			minChunks: 2
		}
	}
}
```

W> This configuration can enlarge your initial bundles, it is recommended to use dynamic imports when a module is not immediately needed.

### Split Chunks: Example 2

Create a `vendors` chunk, which includes all code from `node_modules` in the whole application.

``` js
splitChunks: {
	cacheGroups: {
		commons: {
			test: /[\\/]node_modules[\\/]/,
			name: "vendors",
			chunks: "all"
		}
	}
}
```

W> This might result in a large chunk containing all external packages. It is recommended to only include your core frameworks and utilities and dynamically load the rest of the dependencies.


## `optimization.runtimeChunk`

Setting `optimization.runtimeChunk` to `true` adds an additonal chunk to each entrypoint containing only the runtime.

The value `single` instead creates a runtime file to be shared for all generated chunks.
