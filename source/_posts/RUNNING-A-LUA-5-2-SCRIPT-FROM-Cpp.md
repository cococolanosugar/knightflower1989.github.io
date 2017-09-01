---
title: 如何在c++程序中运行lua脚本
date: 2014-07-16 20:23:09
tags:
  - lua
  - c++
categories:
  - lua
---


原文出处：
[http://www.acamara.es/blog/2012/08/running-a-lua-5-2-script-from-c/](http://www.acamara.es/blog/2012/08/running-a-lua-5-2-script-from-c/)

译文如下：

Lua 是一个胶水语言，可以很方便的去扩展其它宿主语言。说她强大是有很多原因的，而我最喜爱的莫过于她允许无需重新编译即可改变一个程序的执行逻辑。
不幸的是，我发现网上很少有关于lua 5.2的混编教程，而lua 5.2与c++的混编与lua 5.1,lua 5.0,还有lua 4.x是有些差别的，其中包括如下主题：

<!-- more -->

1. c++如何调用Lua 脚本
2. c++与Lua之间如何互传信息
3. Lua如何调用c++
4. Lua如何动态连接c函数库
5. Lua如何实现面向对象
6. 如何在c++和Lua之间互传对象

此文仅是我个人关于上述第一个主题的一些看法。关于后续主题，我将稍后论述。
文中相关代码引用自:
Lua Wiki      [http://lua-users.org/wiki/SampleCode](http://lua-users.org/wiki/SampleCode)
Lua 5.2 手册   [http://www.lua.org/manual/5.2/manual.html](http://www.lua.org/manual/5.2/manual.html)
中文手册       [http://www.photoneray.com/Lua-5.2-Reference-Manual-ZH_CN/#lua_settop](http://www.photoneray.com/Lua-5.2-Reference-Manual-ZH_CN/#lua_settop)

我所使用的Lua为官方版本5.2.1,通过Mac osx10.8.4, gcc4.8进行编译。Lua的编译过程，以及Lua的语法细节，已超出本文范畴，具体详情请参考Lua在线文档。

#### c++调用Lua脚本
这是你能用c++和Lua混编做的最简单的事儿。假如你的Lua脚本的如下：


```lua
-- Simple Hello World Lua program
print('Hello World!')
```


然后，你想从一个简单的c++程序里面调用它。具体步骤如下：

1. 创建Lua状态机--你可以认为是一个运行Lua脚本的虚拟机
2. 加载所需要的Lua库
3. 运行Lua脚本
4. 关闭Lua状态机

因为我们包含了Lua的头文件，故项目必须能够连接预先编译好的Lua库文件，Lua的相关头文件也必须能被项目访问。

完整代码如下所示：
```c++
#include <lua.hpp>
int main(int argc, char* argv[])
{
    // create new Lua state
    lua_State *lua_state;
    lua_state = luaL_newstate();

    // load Lua libraries
    static const luaL_Reg lualibs[] =
    {
        { "base", luaopen_base },
        { NULL, NULL}
    };

    const luaL_Reg *lib = lualibs;
    for(; lib->func != NULL; lib++)
    {
        lib->func(lua_state);
        lua_settop(lua_state, 0);
    }

    // run the Lua script
    luaL_dofile(lua_state, "helloworld.lua");

    // close the Lua state
    lua_close(lua_state);
}
```
程序的输出正如我们所预料的那样：



#### 细节分析
让我们详细的剖析相关代码。
##### 第一段代码是用来创建Lua状态机。

```c++
    // create new Lua state
    lua_State *lua_state;
    lua_state = luaL_newstate();
```


    
代码的意图很明显。唯一要做的就是把通过luaL_newstate()函数创建的Lua状态机的地址保存到一个指针。然后我们就可以通过指针来使用状态机。

##### 第二段代码的作用是加载所需的库文件。
这部分有点难以理解。每个Lua库，包括你自己生成的，都是被c++打开，并保存到Lua状态机的。

```c++
    // load Lua libraries
    static const luaL_Reg lualibs[] =
    {
        { "base", luaopen_base },
        { NULL, NULL}
    };
```

然后，我们把Lua状态机的指针当做参数传给库的加载函数来加载每个库。lua_settop（）函数的的调用是为了确保清除Lua堆栈的所有变量。

```c++
    const luaL_Reg *lib = lualibs;
    for(; lib->func != NULL; lib++)
    {
        lib->func(lua_state);
        lua_settop(lua_state, 0);
    }
```

##### 第三段代码的作用是运行我们的Lua脚本。

```c++
    // run the Lua script
    luaL_dofile(lua_state, "helloworld.lua");
```

你唯一需要提供的就是状态机指针和脚本名称，如果脚本跟c++可执行文件不再同一目录，就加上完整路径。

##### 最后一段代码的作用是关闭状态机。

```c++
    // close the Lua state
    lua_close(lua_state);
```

这会清空状态机所使用的所有内存，除非一些罕见的情况。

以上。
这是最简单的c++和Lua混编。稍后会奉上更加复杂的使用案例，敬请期待。