---
title: 迁移hexo
date: 2016-09-03 00:21:26
categories: 随笔
tags:
    - hexo
---

Blog托管在github，拉取代码后。
- 下载nodejs，[Node.js官网](https://nodejs.org/en/)
- 安装hexo，`sudo npm install -g hexo`
- 保留原有deploy配置，执行`sudo npm install hexo-deployer-git --save`
- 安装server，`sudo npm install hexo-server`

如发生github pages init错误，删除.deploy_git文件
<!-- more -->

theme中配置`_config.yml`文件，将被墙资源改为cdn资源。
另外，360前端cdn资源已经不再提供服务，Google资源服务器已经回到北京，不再需要通过镜像或反向代理等方法获取fonts和jQuery。
目前可以达到秒开博客的效果，Google回归中国似乎指日可待啊。

记录NEXT theme中一个bug：
post_details.js中的方法会导致点击右侧文章目录后，整个目录收起并锚点到某个不可预知的位置，而不是锚点到指定的目录位置。
```
  // 旧方法存在bug，无法选中指定toc
  // $('.post-toc a').on('click', function (e) {
  //   e.preventDefault();
  //   var targetSelector = NexT.utils.escapeSelector(this.getAttribute('href'));
  //   var offset = $(targetSelector).offset().top;

  //   hasVelocity ?
  //     html.velocity('stop').velocity('scroll', {
  //       offset: offset  + 'px',
  //       mobileHA: false
  //     }) :
  //     $('html, body').stop().animate({
  //       scrollTop: offset
  //     }, 500);
  // });
```
修改如下：
```
  $('.post-toc a').on('click', function (e) {
    e.preventDefault();
    $('html, body').animate({
        scrollTop: $( $.attr(this, 'href') ).offset().top
    }, 500);
  });
```

入职百度第一周，充实且兴奋。保持良好工作效率的同时，希望能保持更新博客的习惯。