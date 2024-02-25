


### 安装

* 版本要求
    * node version : 16.X.X
    * hexo version : 6.3.0
过高的版本，hexo 还不兼容，会导致 `hexo g` [生成的静态文件都是空的](https://github.com/zalando-incubator/hexo-theme-doc/issues/149)。

* package
~~~
npm install
~~~

* hexo 
~~~
npm install -g hexo@6.3.0
~~~






### 在 markdown 中使用 asset_img

* npm i -s hexo-asset-link  
* _config.yml 中将 post_asset_folder:false 改为 true  
这样在建立文件时，Hexo会自动建立一个与文章同名的文件夹，这样就可以把与该文章相关的所有资源（图片）都放到那个文件夹里方便后面引用。如这里我放了一张test.jpg的图片。
* `npm install https://github.com/7ym0n/hexo-asset-image --save`  
这是个修改过的插件，经测试无问题

