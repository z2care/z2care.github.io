---
layout: post
title: "Build Your RPB4 Linux With Yocto"
date: 2023-09-03 23:58:00 +0800
categories: 系统构建
tag: Yocto
---
* content
{:toc}

从零开始使用官方环境构建Yocto Linux!

<!-- more -->

### Build Your Raspbarry 4 Linux With Yocto From Zero 

![yocto logo](/docs/assets/images/YoctoProject_Logo_RGB.jpg)

## 一、环境构建
1. 安装和使用官方poky docker即crops/poky
2. sudo docker run --rm -it --mount type=bind,src=/home/xxx/xxx/poky_docker,target=/workdir crops/poky --workdir=/workdir
3. 在/home/xxx/xxx/poky_docker下clone yocto代码，最新LTS是4.12，切换到tag上
4. 在poky_docker下clone处meta-openembedded，引入包仓库，可以使用bblayer-add增加为一个层
5. meta-openembedded不是必需的，可以此刻就更改local.conf编译core-image-minimal

## 二、一些说明
1. 下载的官方poky代码中，meta是一个较完整的qemu、x86等linux系统构建框架，meta-poky中补充了很少一些组件修改，meta-yocto-bsp则提供了实际board bsp支持
2. 如果要用其它的board bsp，则需替换meta-yocto-bsp

## 三、创建board和修改recipe
1. 用bblayer create/add来创建一个meta-xxx
2. 用devtool修改recipe并加回meta-xxx

## 四、最重要的两份参考文档
1. https://docs.yoctoproject.org/singleindex.html
<br>这份文档从使用上介绍
2. https://www.yoctoproject.org/software-overview/
<br>这份文档从宏观上介绍

## 五、其它参考
1. https://github.com/marketplace/actions/build-jekyll-for-github-pages
与https://github.com/actions/deploy-pages配合使用
有范例

2. 注意修改Build步骤的action
Incremental build: disabled. Enable with --incremental

3.

- name: Build and deploy
        uses: helaili/jekyll-action@v2
        with:
          pre_build_commands: git config --global http.version HTTP/1.1; apk fetch git-lfs;
          token: ${{ secrets.GH_TOKEN }}
          target_branch: gh-pages
		  
4. 制作分为两种。
一种是从jekyll源码制作
参考：
https://github.com/Darkade/Darkade.github.io/blob/source/.github/workflows/jekyll.yml


一种是从jekyll blog制作
https://github.com/gruvw/portfolio/blob/master/.github/workflows/build-jekyll.yml

参考：https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
https://github.com/helaili/jekyll-action

5. 后台管理参考
https://github.com/prose/prose

6. 图片加入方法
   可以通过github编辑页面拖拽自动生成
   也可以自己加入到/docs/assets/images
   也可以用图床工具https://github.com/topics/image-hosting
7. 评论系统
   参考https://hutusi.com/articles/comment-via-giscus
8. 搜索系统
   https://cloud.tencent.com/developer/article/1119290
   https://developer.aliyun.com/article/703581
   https://zoharandroid.github.io/2019-08-01-jekyll%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2%E5%AE%9E%E7%8E%B0%E6%90%9C%E7%B4%A2%E5%8A%9F%E8%83%BD/
   https://www.chenshaowen.com/blog/jekyll-search-option.html
   https://blog.csdn.net/dliyuedong/article/details/46848155
