---
title: 在lua中调用c++函数
date: 2014-08-01 21:55:04
tags:
  - lua
  - c++
categories:
  - lua
---

原文出处:[http://www.acamara.es/blog/2012/08/calling-c-functions-from-lua-5-2/](http://www.acamara.es/blog/2012/08/calling-c-functions-from-lua-5-2/)

我的译文如下:

欢迎惠阅我的第三篇Lua5.2的学习笔记。如果你还没读过我的前两篇文章，敬请移步[如何在c++程序中运行lua脚本](/2014/07/16/RUNNING-A-LUA-5-2-SCRIPT-FROM-Cpp/)，  [从lua脚本中向c++传递变量](/2014/07/25/PASSING-VARIABLES-FROM-LUA-5-2-TO-Cpp/)。

本篇新教程讲讲述如何在Lua5.2中给c++函数传递参数，并接收返回值。
文中大多引述均来自lua用户WIKI和lua5.2的用户手册。

嗯,我还是那个一如既往帅气阳光的基督小伙儿 Stigen Larsen。


首先，我将展示Lua和c++代码，以及其运行的输出结果。然后逐步剖析其中难点。

<!-- more -->

#### Lua 脚本部分

Lua脚本将向c++函数传递一些参数，然后打印出c++返回的两个参数。
代码如下：

```lua
-- call the registered C-function
io.write('[Lua] Calling the C functionn')
a,b = displayLuaFunction(12, 3.141592, 'hola')

-- print the return values
io.write('[Lua] The C function returned <' .. a .. '> and <' .. b .. '>n')
```

#### c++部分


c++代码和前文基本一致。区别之处就是声明函数和将函数压入Lua堆栈的部分。
完整代码如下:

```c++
#include <iostream>
#include <sstream>

#include <lua.hpp>

int displayLuaFunction(lua_State*);

int main(int argc, char* argv[])
{
    // new Lua state
    std::cout << "[C++] Starting Lua state" << std::endl;
    lua_State *lua_state = luaL_newstate();

    // load Lua libraries
    std::cout << "[C++] Loading Lua libraries" << std::endl;
    static const luaL_Reg lualibs[] = 
    {
        {"base", luaopen_base},
        {"io", luaopen_io},
        {NULL, NULL}
    };
    const luaL_Reg *lib = lualibs;
    for(; lib->func != NULL; lib++)
    {
        std::cout << " loading '" << lib->name << "'" << std::endl;
        luaL_requiref(lua_state, lib->name, lib->func, 1);
        lua_settop(lua_state, 0);
    }

    // push the C++ function to be called from Lua
    std::cout << "[C++] Pushing the C++ function" << std::endl;
    lua_pushcfunction(lua_state, displayLuaFunction);
    lua_setglobal(lua_state, "displayLuaFunction");

    // load the script
    std::cout << "[C++] Loading the Lua script" << std::endl;
    int status = luaL_loadfile(lua_state, "parrotscript.lua");
    std::cout << " return: " << status << std::endl;

    // run the script with the given arguments
    std::cout << "[C++] Running script" << std::endl;
    int result = 0;
    if(status == LUA_OK)
    {
        result = lua_pcall(lua_state, 0, LUA_MULTRET, 0);
    }
    else
    {
        std::cout << " Could not load the script." << std::endl;
    }

    // close the Lua state
    std::cout << "[C++] Closing the Lua state" << std::endl;
    return 0;
}

int displayLuaFunction(lua_State *l)
{
    // number of input arguments
    int argc = lua_gettop(l);

    // print input arguments
    std::cout << "[C++] Function called from Lua with " << argc 
              << " input arguments" << std::endl;
    for(int i=0; i<argc; i++)
    {
        std::cout << " input argument #" << argc-i << ": "
                  << lua_tostring(l, lua_gettop(l)) << std::endl;
        lua_pop(l, 1);
    }

    // push to the stack the multiple return values
    std::cout << "[C++] Returning some values" << std::endl;
    lua_pushnumber(l, 12);
    lua_pushstring(l, "See you space cowboy");

    // number of return values
    return 2;
}
```


#### 详细剖析

函数在Lua中是一等公民。这意味着Lua函数可以像普通变量一样存储。我们可以像对待普通变量一样把函数压入Lua堆栈，然后命名为全局变量。
```c++
    // push the C++ function to be called from Lua
    lua_pushcfunction(lua_state, displayLuaFunction);
    lua_setglobal(lua_state, "displayLuaFunction");
```

Lua就可以通过这个全局变量访问我们的函数displayLuaFunction()。除此之外，还有一个宏lua_register()，可以把上述步骤合二为一。
所以上述代码等价于：

```c++
    // push the C++ function to be called from Lua
    lua_register(lua_state, "displayLuaFunction", displayLuaFunction);
```

（感谢kpityu小伙儿在评论种指出这点）

为了便于c++和Lua交换数据，代码必须遵循一些规则。

##### 规则NO1：函数的定义和lua_CFunction一致

```c++
typedef int (*lua_CFunction) (lua_State *L);
```

即函数必须接收指向lua_State的指针作为参数，并且返回整形。

##### 规则NO2:遵守Lua和c++交换数据的协议。
正如你所见，所有数据都是通过Lua堆栈作为桥梁交换的。
例如，当Lua调用c++函数时
```c++
displayLuaFunction(a, b, c)
```
Lua会为此函数创建新的堆栈，该堆栈独立于其他堆栈。堆栈种包含输入参数，因此呢，我们在c++函数中可以通过Lua堆栈来获取所有参数。
```c++
    // number of input arguments
    int argc = lua_gettop(l);
```
并且可以直接获取
```c++
    lua_tostring(l, lua_gettop(l));
```

需要注意的是，参数按传参顺序入栈，逆序出栈。由于一开始堆栈中只有输入参数，你可以直接通过索引获取其值。如果第二个参数可以被转化成字符串，我们可以这样获取它：
```c++
    lua_tostring(l, 2);
```
一旦该函数执行完毕，其返回值必须放入堆栈以便Lua使用它们。
```c++
    // push to the stack the multiple return values
    lua_pushnumber(l, 12);
    lua_pushstring(l, "See you space cowboy");
```
最后，为了让Lua能够获取c++回传的数据，我们需要返回放入Lua堆栈的变量的数量
```lua
    // number of return values
    return 2;
```
是的，就是这么简单！

此文讲述了Lua和c++混编的基本协议。如果想查看更多示例，请查阅Lua的相关库，它们都是严格遵循这些规则的。

我在犹豫，下篇文章是讨论Lua和c++混编时动态链接库的使用，还是讨论Lua和c++之间对象的传递呢，敬请期待。
