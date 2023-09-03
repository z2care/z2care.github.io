---
layout: original
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
## 一、环境构建
# 1. 安装和使用官方poky docker即crops/poky
# 2. sudo docker run --rm -it --mount type=bind,src=/home/xxx/xxx/poky_docker,target=/workdir crops/poky --workdir=/workdir
# 3. 在/home/xxx/xxx/poky_docker下clone yocto代码，最新LTS是4.12，切换到tag上
# 4. 在poky_docker下clone处meta-openembedded，引入包仓库，可以使用bblayer-add增加为一个层
# 5. meta-openembedded不是必需的，可以此刻就更改local.conf编译core-image-minimal

## 二、一些说明
# 1. 下载的官方poky代码中，meta是一个较完整的qemu、x86等linux系统构建框架，meta-poky中补充了很少一些组件修改，meta-yocto-bsp则提供了实际board bsp支持
# 2. 如果要用其它的board bsp，则需替换meta-yocto-bsp

## 三、创建board和修改recipe
# 1. 用bblayer create/add来创建一个meta-xxx
# 2. 用devtool修改recipe并加回meta-xxx

## 四、最重要的两份参考文档
# 1. https://docs.yoctoproject.org/singleindex.html
<br>这份文档从使用上介绍
# 2. https://www.yoctoproject.org/software-overview/
<br>这份文档从宏观上介绍
