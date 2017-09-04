---
title: cocos2dx 3.13.1 jsbinding 测试记录
date: 2016-11-16 11:08:33
tags:
  - cocos2dx
  - jsb
categories:
  - cocos2dx
---

<center>
cocos2dx 3.13.1 jsbinding 测试记录
</center>

<!-- more -->

1. cocos new -p com.mz.jsbtest -d ./ -l js  JSBTest

2. 在cocos2dx/cocos目录新建目录my 增加类文件

    ```c++
    //CustomClass.hpp
    #ifndef CustomClass_hpp
    #define CustomClass_hpp

    #include <stdio.h>
    #include <iostream>

    #include "cocos2d.h"
    USING_NS_CC;
    namespace cocos2d {
        class CustomClass : public cocos2d::Ref
        {
        public:
            
            CustomClass();
            
            ~CustomClass();
            
            bool init();
            
            std::string helloMsg();
            
            CREATE_FUNC(CustomClass);
        };
    } //namespace cocos2d

    #endif /* CustomClass_hpp */
    ```


    ```c++
    //CustomClass.cpp
    #include "CustomClass.hpp"

    USING_NS_CC;

    CustomClass::CustomClass(){
        
    }

    CustomClass::~CustomClass(){
        
    }

    bool CustomClass::init(){
        return true;
    }

    std::string CustomClass::helloMsg() {
        return "Hello from CustomClass::sayHello";
    }
    ```
3. 在cocos2dx/tools/tojs 目录下新增cocos2dx_custom.ini配置文件用于描述绑定的名字空间，前缀等信息，如果提示错误尽量参考tojs目录下的其他ini文件的格式如cocos2dx_builder.ini，不同cocos2dx版本之间语法貌似有差异,注意这里我们新增的类继承了Ref类，所以base_classes_to_skip = Ref，否则会提示Ref x86_64未定义啥的，如果是纯是自己写的类就不用了

    ```text
    [cocos2dx_custom]
    name = cocos2dx_custom
    prefix = cocos2dx_custom
    target_namespace = cc

    headers = %(cocosdir)s/cocos/my/CustomClass.hpp
    classes = CustomClass

    android_headers = -I%(androidndkdir)s/platforms/android-14/arch-arm/usr/include -I%(androidndkdir)s/sources/cxx-stl/gnu-libstdc++/4.8/libs/armeabi-v7a/include -I%(androidndkdir)s/sources/cxx-stl/gnu-libstdc++/4.8/include -I%(androidndkdir)s/sources/cxx-stl/gnu-libstdc++/4.9/libs/armeabi-v7a/include -I%(androidndkdir)s/sources/cxx-stl/gnu-libstdc++/4.9/include
    android_flags = -D_SIZE_T_DEFINED_ 

    clang_headers = -I%(clangllvmdir)s/%(clang_include)s 
    clang_flags = -nostdinc -x c++ -std=c++11 -U __SSE__

    cocos_headers = -I%(cocosdir)s -I%(cocosdir)s/cocos -I%(cocosdir)s/cocos/editor-support -I%(cocosdir)s/cocos/platform/android -I%(cocosdir)s/external

    cocos_flags = -DANDROID

    extra_arguments = %(android_headers)s %(clang_headers)s %(cxxgenerator_headers)s %(cocos_headers)s %(android_flags)s %(clang_flags)s %(cocos_flags)s %(extra_flags)s 

    classes_have_type_info = no

    # base classes which will be skipped when their sub-classes found them.
    base_classes_to_skip = Ref

    # Determining whether to use script object(js object) to control the lifecycle of native(cpp) object or the other way around. Supported values are 'yes' or 'no'.
    script_control_cpp = yes
    ```
4. 在tojs目录下，修改执行genbindings.py文件尽量注释掉cmd_args中cocos2dx自带的那些绑定，再加入自己定制的绑定，否则容易出错，就呆重新建项目， 然后执行python genbindings.py
也可以加在custom_cmd_args里面会生成到cocos2dx/frameworks/custome/auto，不过目录处理起来比较麻烦
5. 生成的代码在cocos2dx/cocos/scripting/js-bindings/auto下面，加入Xcode中的cocos2d_js_bindings.xcodeproj
my目录加到头文件搜索路径
6. 在AppDelegate.cpp引入生成的头文件

    ```c++
    #include "scripting/js-bindings/auto/jsb_cocos2dx_custom.hpp"
    ```

    在applicationDidFinishLaunching函数加入
    sc->addRegisterCallback(register_all_cocos2dx_custom);
    然后就可以在app.js中调用我们绑定的代码了

    ```cpp
    var customClass = cc.CustomClass.create();
    var msg = customClass.helloMsg()
    cc.log("customClass's msg is : " + msg)
    ```