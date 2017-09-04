---
title: 从论坛下载的老版本cocos2dx项目怎么运行
date: 2016-10-19 10:46:59
tags:
  - cocos2dx
  - oc
categories:
  - cocos2dx
---

从论坛下载的cocos2dx项目，需要研究其逻辑，或者做二次开发，经常发现不能直接运行，以下是一些经验性的解决方法。

<!-- more -->

从网上下载的cocos2dx项目运行或升级步骤：

1. 下载xxx.zip文件，解压
2. 若是3.x的可以直接运行，2.x的如果不带库文件不能直接运行，需要放到cocos2dx-2.x.x的projects目录下面运行测试
3. 如果放到projects文件夹下面可以正常运行，则进入如下步骤：
    1. 创建SVN目录树
    2. 将项目放入trunk目录
    3. 将cocos2dx引擎的四个文件夹 coco2dx CocosDenshion extensions external 拷贝到trunk目录
    4. 打开 ios项目文件 修改项目目录树里面的cocos2dx.xcodeproj Box2d chipmunk CocosDenshion extensions libwebsocket 指向的上述拷贝过来的对应文件夹
    5.  修改项目 build setting-> search paths  改为当前相对目录，如上即使依次删除 header search paths和library search paths 各个路径的“../../”
    6. 修改build phases->link binary with libraries 增加libcocos2dx.a
    7. 打开mac版项目文件 做上述同样步骤的修改
    8. Android项目修改对应的 Android.mk文件索引对应的代码文件，build_native.sh修改COCOS2DX_ROOT



osx系统升级，会遇到cocos2dx 2.x项目 如果遇到
```objc
m_pValueDict->setObject( CCString::create( (const char*)glGetString(GL_VENDOR)), "gl.vendor");
```
报错

修改 mac项目的 AppController.mm文件的 applicationDidFinishLaunching函数
在
```objc
glView = [[EAGLView alloc] initWithFrame:rect pixelFormat:pixelFormat];
```
下面加入
```objc
[glView prepareOpenGL];
```



方案一：
```objc
        NSOpenGLContext *ctx = [[NSOpenGLContext alloc] initWithFormat:pixelFormat shareContext:nil];
        if(ctx)
        {
            [glView setOpenGLContext:ctx];
            [ctx setView:glView];
            [ctx makeCurrentContext];
        }
```
方案二：
```objc
        [glView prepareOpenGL];
```