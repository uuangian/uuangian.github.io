title: "Windows下建站记录 hexo"
date: 2015-04-17 09:56:52
tags:
---

## 应用安装
**安装 node.js**

- 将其路径 \nodejs\ 加入环境变量，可以在命令框中使用其命令
- 安装npm 
   这个工具应该是个插件的管理工具，后面安装插件时会依赖它，所以要先把它安装好。
   默认node.js会安装npm
- 安装Hexo
```
npm install -g hexo-cli #-g 是全局安装
```
**安装 Git**
Windows 下 
msysgit
tortoisegit 可选
  > 之后执行hexo或git命令时都在Git Bash中操作

## Hexo 建站

**初始化** *[详见](http://hexo.io/docs/setup.html)*
```
hexo init <folder>
cd <folder>
npm install
```
**发文** *[详见](http://hexo.io/docs/writing.html)*
```
hexo new <title>
```
**生成** *[详见](http://hexo.io/docs/generating.html)*
```
hexo g
```
如果重新生成
```
hexo clean 
hexo g
```
**本地服务** *[详见](http://hexo.io/docs/generating.html)*
```    
hexo server
```
## Hexo 设置相关
Hexo的教程网上有许多，收集一些如下

- [使用hexo搭建博客](http://yangjian.me/workspace/building-blog-with-hexo/)
- [Hexo-的学习、搭建、整理](http://lszb811.com/2014/07/08/Hexo-%E7%9A%84%E5%AD%A6%E4%B9%A0%E3%80%81%E6%90%AD%E5%BB%BA%E3%80%81%E6%95%B4%E7%90%86/)
- [hexo系列教程](http://zipperary.com/2013/05/28/hexo-guide-1/)

**1. Markdown**

- 默认hexo-renderer-marked 并目前不支持角标功能，替换使用 [hexo-renderer-markdown-it](https://github.com/celsomiranda/hexo-renderer-markdown-it)
开启角标功能及其他参数设置在全局配置文件_config.yml中
```
markdown:
  render:
    html: true # 可以保持html格式
    xhtmlOut: false
    breaks: false
    linkify: true
    typographer: true
    quotes: '“”‘’'
  plugins:
    - markdown-it-footnote
    - markdown-it-sub
    - markdown-it-sup
```
- [markdown 常用语法](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)
**2. FontAwesome**

- 简洁的图标型字体库 [<i class="icon-html5"></i>Font-Awesome](http://fortawesome.github.io/Font-Awesome)
- 配置在主题目录themes\xx\source\css\fonts\下
  在css\_variables.styl中可定义字体名称及路径
```
// Fonts
font-icon = FontAwesome
font-icon-path = "fonts/fontawesome-webfont"
font-icon-version = "4.0.3"
```
  在css\style.styl文件中定义字体加载，此文件定义了全局的css格式，在其中声明可以应用到整个网站
```
@font-face
  font-family: FontAwesome
  font-style: normal
  font-weight: normal
  src: url(font-icon-path + ".eot?v=#" + font-icon-version)
  src: url(font-icon-path + ".eot?#iefix&v=#" + font-icon-version) format("embedded-opentype"),
       url(font-icon-path + ".woff?v=#" + font-icon-version) format("woff"),
       url(font-icon-path + ".ttf?v=#" + font-icon-version) format("truetype"),
       url(font-icon-path + ".svg#fontawesomeregular?v=#" + font-icon-version) format("svg")
```
 在\layout\_partial\head.ejs中设置其css文件路径，可以使用在线地址或本地路径。
 本地路径放置在\css\font-awesome.min.css下，路径设置如下。当需要使用最新或指定版本的css文件时，本地存放更为方便。
 ```
 <head>
...
...
<%- css('css/font-awesome.min') %>
...
</head>
 ```