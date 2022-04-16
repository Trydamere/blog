## blog

#### 创建文章
执行如下命令创建一篇新文章，名为《测试文章》

```
hexo new post 测试文章
```


执行完成后在`source\_posts`目录下生成了一个md文件和一个同名的资源目录(用于存放图片)

在资源目录测试文章中放一张图片 test.png

在测试文章.md中添加内容如下，演示了图片的三种引用方式。第一种为官方推荐用法，第二种为markdown语法，第三种和前两种图片存放位置不一样，是将图片放在`\source\images`目录下。这三种写法在md文件中图片是无法显示的，但是在页面上能正常显示。

图片的引入方式可参考官方文档 https://hexo.io/zh-cn/docs/asset-folders.html

```
{% asset_img test.png 图片引用方法一 %}

![图片引用方法二](test.png)

![图片引用方法三](/images/test.png)
```


本地启动

```
hexo g -d
hexo s
```

浏览器访问 http://localhost:4000
文章添加成功，之后push到远程仓库，完成发布。
