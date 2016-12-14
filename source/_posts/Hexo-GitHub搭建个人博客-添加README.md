---
title: Hexo GitHub搭建个人博客-添加README
date: 2016-07-11 18:05:12
tags: 开发工具
---
# 概要

当我们使用Hexo+Github搭建好个人博客环境之后，默认Github上repository中是没有Readme的。本文主要介绍如何添加Readme。

<!-- more -->

# 如何添加Readme

如果我们直接在Github上增加Readme文件，在执行了hexo d部署命令之后，Readme文件会自动消失。

# 解决办法
将我们要添加的Readme.md文件放入本地hexo工作目录的source根目录下，然后修改_config.yml文件，在skip_render后增加我们添加的文件名README.md：

```
skip_render: [README.md](http://README.md)  
```

需要注意的是，冒号后一定要有空格， 否则运行hexo g命令的时候会出现莫名其妙的错误：

```
xiajundeMacBook-Pro:hexo xiajun$ sudo hexo d  
FATAL can not read a block mapping entry; a multiline key may not be an implicit key at line 30, column 1:  
    # Writing  
    ^  
YAMLException: can not read a block mapping entry; a multiline key may not be an implicit key at line 30, column 1:  
    # Writing  
    ^  
    at generateError (/Users/xiajun/hexo/node_modules/hexo/node_modules/js-yaml/lib/js-yaml/loader.js:162:10)  
    at throwError (/Users/xiajun/hexo/node_modules/hexo/node_modules/js-yaml/lib/js-yaml/loader.js:168:9)  
    at readBlockMapping (/Users/xiajun/hexo/node_modules/hexo/node_modules/js-yaml/lib/js-yaml/loader.js:1040:9)  
    at composeNode (/Users/xiajun/hexo/node_modules/hexo/node_modules/js-yaml/lib/js-yaml/loader.js:1326:12)  
    at readDocument (/Users/xiajun/hexo/node_modules/hexo/node_modules/js-yaml/lib/js-yaml/loader.js:1488:3)  
    at loadDocuments (/Users/xiajun/hexo/node_modules/hexo/node_modules/js-yaml/lib/js-yaml/loader.js:1544:5)  
    at Object.load (/Users/xiajun/hexo/node_modules/hexo/node_modules/js-yaml/lib/js-yaml/loader.js:1561:19)  
    at Hexo.yamlHelper (/Users/xiajun/hexo/node_modules/hexo/lib/plugins/renderer/yaml.js:7:15)  
    at Hexo.tryCatcher (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/util.js:16:23)  
    at Hexo.&lt;anonymous&gt; (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/method.js:15:34)  
    at /Users/xiajun/hexo/node_modules/hexo/lib/hexo/render.js:59:21  
    at tryCatcher (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/util.js:16:23)  
    at Promise._settlePromiseFromHandler (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/promise.js:502:31)  
    at Promise._settlePromise (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/promise.js:559:18)  
    at Promise._settlePromise0 (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/promise.js:604:10)  
    at Promise._settlePromises (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/promise.js:683:18)  
    at Async._drainQueue (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/async.js:138:16)  
    at Async._drainQueues (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/async.js:148:10)  
    at Immediate.Async.drainQueues [as _onImmediate] (/Users/xiajun/hexo/node_modules/hexo/node_modules/bluebird/js/release/async.js:17:14)  
    at processImmediate [as _immediateCallback] (timers.js:368:17)  
```

然后我们执行部署命令部署到github，就可以看到repository中有了Readme文件，并且下次再次部署的时候，依然存在。

关于skip_render的含义，官方文档解释是：  
跳过指定文件的渲染，您可使用 glob 表达式来匹配路径。

# 参考链接

[How can I tell Hexo to ignore a file when creating posts?-StackOverflow](http://stackoverflow.com/questions/31494145/how-can-i-tell-hexo-to-ignore-a-file-when-creating-posts)  
[Hexo \_posts目录, skip_render 的说明](http://www.yczmm.com/hexo-skip_render.html)  
[Hexo官方文档](https://hexo.io/zh-cn/docs/configuration.html)