##Snowpack hmr实现思路

###Snowpack是什么？
	
 [Snowpack](https://github.com/pikapkg/snowpack)是一个快速构建现代web应用的工具，它在应用程序中利用esm导入避免了开发阶段的构建工作。构建时立即启动(50-ms)，并且每次更改都不会浪费时间重新构建，可以在浏览器中立即看到改动。现阶段它支持了市面上比较通用的技术库（react\vue\ Svelte 等）和工具（Babel、ts、PostCSS等）。
	
###Snowpack 和 Webpack 的区别

直白的讲Webpack是一个打包器，应用程序中每一个文件都是一个webpack的模块，webpack从指定入口文件分析经过各种loader将不同类型的文件处理成js函数包装后的模块并将预编译语言转换成浏览器识别的语言，再经过各种plugin进行文件拆分、打包和性能优化等，处理过程中建立文件的依赖关系。
而snowpack产出的文件(无构建)保持满足esm模块的代码不变，建立文件的依赖关系。

### HMR

####webpack hmr

 我们先看看[webpack hmr](https://juejin.im/post/5d145d4e6fb9a07eee5ededa)，简单点说每次文件变化webpack找到文件变化以后引起了哪些资源变化，然后打出差异包和变更hash通过socket推送给浏览器，然后浏览器收到信息以后去加载hash.hot-update.json, json返回变更file_name key，根据 file_name.hash.hot-update.js 加载打包好的文件live reload加载更新文件。
 
![webpack hmr流程图](https://cdn.nlark.com/yuque/0/2020/png/103868/1591932480018-79868dbf-0bec-472e-9c48-4c6d4a6eef32.png)
（图片来源于[这里](https://juejin.im/post/5d145d4e6fb9a07eee5ededa)）

####snowpack如何实现的？

snowpack模块热替换是基于esm-hmr实现的流程如下:
	![流程](https://cdn.nlark.com/yuque/0/2020/png/103868/1591934287325-f4a90616-38aa-49d8-a03e-5fa9e107090b.png)

####过程分析：

1、监听文件变化以后，snowpack会将变化文件依赖关系的文件动态的增加时间戳尾巴；

	![更改文件](https://cdn.nlark.com/yuque/0/2020/png/103868/1591934418387-83d0e1df-b393-4735-8f16-14b0caa17297.png)
	
2、esm-hmr推送一条指令给socket client

![socket消息](https://cdn.nlark.com/yuque/0/2020/png/103868/1591934468025-0db91db7-f1be-4ff8-8636-5d1cc81e075a.png)

3、浏览器端监听socket，根据类型最终调用applyUpdate('/_dist_/index.js'), 并动态生成一个import加载变更文件和依赖资源

```
	async function applyUpdate(id) {
  const state = REGISTERED_MODULES[id];
  if (!state) {
    return false;
  }
  if (state.isDeclined) {
    return false;
  }
  const acceptCallbacks = state.acceptCallbacks;
  const disposeCallbacks = state.disposeCallbacks;
  state.disposeCallbacks = [];
  state.data = {};
  disposeCallbacks.map((callback) => callback());
  const updateID = Date.now();
  for (const {deps, callback: acceptCallback} of acceptCallbacks) {
    const [module, ...depModules] = await Promise.all([
      import(id + `?mtime=${updateID}`),
      ...deps.map((d) => import(d + `?mtime=${updateID}`)),
    ]);
    acceptCallback({module, deps: depModules});
  }
  return true;
}

socket.addEventListener('message', ({data: _data}) => {
  if (!_data) {
    return;
  }
  const data = JSON.parse(_data);
  debug('message', data);
  if (data.type === 'reload') {
    debug('message: reload');
    reload();
    return;
  }
  if (data.type !== 'update') {
    debug('message: unknown', data);
    return;
  }
  debug('message: update', data);
  debug(data.url, Object.keys(REGISTERED_MODULES));
  applyUpdate(data.url)
    .then((ok) => {
      if (!ok) {
        reload();
      }
    })
    .catch((err) => {
      console.error(err);
      reload();
    });
});
```

4、加载esm的代码中存在变化的文件自动打上了时间戳尾巴，浏览器根据url决定是否重新请求

![加载新模块](https://cdn.nlark.com/yuque/0/2020/png/103868/1591939445110-9cf194c6-2c9d-4bd2-87f4-4e9e4177059a.png?x-oss-process=image%2Fresize%2Cw_1492)

