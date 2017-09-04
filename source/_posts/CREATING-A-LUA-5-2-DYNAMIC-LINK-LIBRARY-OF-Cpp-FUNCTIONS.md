---
title: 用lua 5.2为c++创建动态链接库
date: 2014-09-07 22:12:49
tags:
  - lua
  - c++
categories:
  - lua
---

原文链接：[http://www.acamara.es/blog/2012/08/creating-a-lua-5-2-dynamic-link-library-of-c-functions/](http://www.acamara.es/blog/2012/08/creating-a-lua-5-2-dynamic-link-library-of-c-functions/)

我的译文如下：


欢迎惠阅我的第四篇c++和Lua5.2混编的学习笔记。如果你还没读过我的前三篇文章，敬请移步[如何在c++程序中运行lua脚本](/2014/07/16/RUNNING-A-LUA-5-2-SCRIPT-FROM-Cpp/)，  [从lua脚本中向c++传递变量](/2014/07/25/PASSING-VARIABLES-FROM-LUA-5-2-TO-Cpp/)， [在lua中调用c++函数](/2014/08/01/CALLING-Cpp-FUNCTIONS-FROM-LUA-5-2/)。


本篇新教程将讲述如何创建c动态链接库，以及如何在Lua5.2中使用它们。
我将假设你已经懂得怎么创建动态链接库。本文代码将在通过Mac osx10.8.4, gcc4.8进行编译。你可以略做修改转换编译器，然后运行到其他系统。
文中大多引述均来自lua用户WIKI和lua5.2的用户手册。

<!-- more -->

#### 本文目的

假设你已经有一些有用的并且能和Lua5.2适配的c或者c++函数（记住必须是lua_CFunction类型的函数哦）。为了重用代码，你希望将它们打包成库，然后再需要的时候动态链接。就如同你使用Lua标准库一样。
Lua标准库中有io, string, table等等，在需要的时候可以加载它们。

本文，我们将学习怎样创建C语言动态链接库，并在Lua中调用它们。

由于本文使用的c++代码基本与前文一致，所以就省略不写了。但是会展示简短的Lua代码以便更好的理解程序的输出结果，以及将要创建C语言动态链接库的c++代码。
你可以从个链接下载这三部分代码。

#### Lua脚本部分

Lua脚本将动态的链接我们的新的库，其名字为cfunctions。
```lua
-- load the c-function library
io.write("[Lua] Loading the C-function libraryn")
cfunctions = require("cfunctions")

-- use some of the functions defined in the library
io.write("[Lua] Executing the C-library functionsn");
local f = cfunctions.fun1()
io.write("[Lua] Function 1 says it's ", f, " times funnyn"); 
f = cfunctions.fun2()
io.write("[Lua] Function 2 says it's ", f, " times funnyn"); 
io.write("[Lua] Exiting Lua scriptn")
```

库文件代码如下：
```c++
#include <iostream> 
#include <lua.hpp>

/*
 * Library functions
 */

extern "C"
{

static int funval = 10;

static int function_1(lua_State* l)
{
    std::cout << "[DLL] Function 1 is very funny." << std::endl;
    std::cout << "[DLL] It's fun value is: " << funval << std::endl;
    lua_pushnumber(l, funval);
    return 1;
}

static int function_2(lua_State* l)
{
    std::cout << "[DLL] Function 2 is twice as funny!" << std::endl;
    std::cout << "[DLL] It's fun value is: " << 2*funval << std::endl;
    lua_pushnumber(l, 2*funval);
    return 1;
}

/*
 * Registering functions
 */

static const struct luaL_Reg cfunctions[] = {
    {"fun1", function_1},
    {"fun2", function_2},
    {NULL, NULL}
};

int __declspec(dllexport) luaopen_cfunctions(lua_State* l)
{
    luaL_newlibtable(l, cfunctions);
    luaL_setfuncs(l, cfunctions, 0);
    return 1;
}
} // end extern "C"
```

#### 细节分析

用于产生动态链接库的代码包含一个导出函数luaopen_cfunctions()和两个隐藏的函数function_1()，function_2()。导出函数会将两个隐藏函数注册到Lua。这样，就不能从外部直接调用它们了。

为了注册该库，我们首先创建一个带有需要注册函数的信息的结构体。在我们的例子中就是luaL_Reg结构体的数组cfunctions：

```c++
static const struct luaL_Reg cfunctions[] = {
    {"fun1", function_1},
    {"fun2", function_2},
    {NULL, NULL}
};
```

需要说明的时，此例中在Lua将用fun1,fun2分别代替function_1，function_2。然后用luaL_newlibtable()在堆栈顶部创建library table，并用luaL_setfuncs()注册我们的函数。

```c++
    luaL_newlibtable(l, cfunctions);
    luaL_setfuncs(l, cfunctions, 0);
```

最后，返回加入堆栈的元素个数，只有一个，就是我们的library table。

#### 咦，怎么报错了！！！

可能有许多种原因，据我所知主要有以下两种：

如果报错信息是“The specified procedure could not be found”（找不到指定的程序）就很容易解决。我认为弹出这个错误的原因主要有两个：

##### 其一，你忘记了用extern C 语句包含库文件代码。
Lua底层用的是C语言，不能被c++链接。所以要告诉编译器用C语言的编译规则去编译库文件。解决方法就是用extern C包含你的库代码。

##### 其二，你忘记了导出你的函数。
即调用luaopen_cfunctions()声明你要导出的函数。在微软的编译器上，你还必须早导出函数前面加上__declspecs(dllexport)宏，
在其他的编译器上，仅仅定义为extern函数就行了。否则，你就是把自己的函数隐藏了，当然，编译器不会给你好脸色的。

如果报错信息是“Multiple Lua VMs detected”（检测到多个Lua虚拟机），说明你遇到了很隐蔽的错误。哥只有一个Lua状态机，所以第一次看到这个错误信息，哥被迷惑了。
你不能在Lua里面同时静态链接c++宿主程序和C库，你必须动态的往Lua库中链接它们。至少我这样解决了这个问题。

以上。

希望你已经学会怎么创建C函数库，并能通过require()函数动态的链接到Lua5.2。这与在c++中注册函数供Lua使用，并没有什么不同，但是却能大大提高代码的重用率。

稍后将奉上在c++和Lua中互传对象的教程，也即是我们的最终教程。敬请期待。