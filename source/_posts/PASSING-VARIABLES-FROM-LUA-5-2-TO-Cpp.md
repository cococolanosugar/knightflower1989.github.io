---
title: 从lua脚本中向c++传递变量
date: 2014-07-25 21:37:44
tags:
  - lua
  - c++
categories:
  - lua
---


原文出处：[http://www.acamara.es/blog/2012/08/passing-variables-from-lua-5-2-to-c-and-vice-versa/](http://www.acamara.es/blog/2012/08/passing-variables-from-lua-5-2-to-c-and-vice-versa/)

我的译文如下

这是第二篇关于我和c++lua混编肉搏的笔记。如果你还没读过我的第一篇文章，敬请移步[如何在c++程序中运行lua脚本](/2014/07/16/RUNNING-A-LUA-5-2-SCRIPT-FROM-Cpp/)。
文中大多引述均来自lua用户WIKI和lua5.2的用户手册。
上篇文章仅仅只是抛砖引玉，c++和lua混编的强悍之处我们并未深入介绍。我希望通过以下示例让你能更进一步了解和使用lua.
我们的目标有两个：第一，从宿主语言c++往Lua脚本中传递参数。第二，从Lua脚本中接收返回数据。注意，Lua脚本可以返回不止一个数据哦！

<!-- more -->

#### Lua脚本部分

以下Lua脚本只是把我们存入全局的arg序列中的数据打印我们出来，然后返回不同类型Lua变量。
代码如下：

```lua
-- print the arguments passed from C
io.write("[Lua] These args were passed into the script from Cn")
for i=1,#arg do
    print("      ", i, arg[i])
end 
-- return a value of different Lua types (boolean, table, numeric, string)
io.write("[Lua] Script returning data back to Cn")

-- create the table
local temp = {}
temp[1]=9
temp[2]="See you space cowboy!"

return true,temp,21,"I am a mushroom"
```

为了在c++和Lua之间互传信息，我们必须编译c++程序。

#### c++部分
正如上篇教程，我会先奉上完整代码，然后逐步剖析。

```c++
#include <iostream>
#include <sstream>
#include <lua.hpp>

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

    // start the arg table in Lua
    std::cout << "[C++] Creating the arg table" << std::endl;
    lua_createtable(lua_state, 2, 0);
    lua_pushnumber(lua_state, 1);
    lua_pushnumber(lua_state, 49);
    lua_settable(lua_state, -3);
    lua_pushnumber(lua_state, 2);
    lua_pushstring(lua_state, "Life is a beach");
    lua_settable(lua_state, -3);
    lua_setglobal(lua_state, "arg");

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

    // print the values returned from the script
    std::cout << "[C++] Values returned from the script:" << std::endl;
    std::stringstream str_buf;
    while(lua_gettop(lua_state))
    {
        str_buf.str(std::string());
        str_buf << " ";
        switch(lua_type(lua_state, lua_gettop(lua_state)))
        {
        case LUA_TNUMBER:
            str_buf << "script returned the number: "
                 << lua_tonumber(lua_state, lua_gettop(lua_state));
            break;
        case LUA_TTABLE:
            str_buf << "script returned a table";
            break;
        case LUA_TSTRING:
            str_buf << "script returned the string: "
                 << lua_tostring(lua_state, lua_gettop(lua_state));
            break;
        case LUA_TBOOLEAN:
            str_buf << "script returned the boolean: "
                 << lua_toboolean(lua_state, lua_gettop(lua_state));
            break;
        default:
            str_buf << "script returned an unknown-type value";
        }
        lua_pop(lua_state, 1);
        std::cout << str_buf.str() << std::endl;
    }

    // close the Lua state
    std::cout << "[C++] Closing the Lua state" << std::endl;
    return 0;
}
```

正如亲眼所见，程序的基本结构一如从前。我们打开一个状态机，加载库文件，在状态机上运行脚本，最后关闭状态机。
程序输出如下：

#### 细节剖析

细心如你者，一定发现了这次我们并未使用luaL_dofile()函数（事实上它是一个宏）来执行脚本。而是代之以luaL_loadfile()加载脚本，以lua_pcall()执行脚本。这是为了展示luaL_dofile宏的原理，实际上luaL_dofile是以上两个函数的合体，其宏定义如下：

```c++
    (luaL_loadfile(L, filename) || lua_pcall(L, 0, LUA_MULTRET, 0))
```

我认为是时候，介绍一下Lua堆栈了。Lua堆栈是连接宿主语言和Lua的桥梁。宿主语言可以在堆栈中放入Lua状态机可见的数据，而Lua状态机也可以在Lua堆栈中放入宿主语言可见的数据。虽然听起来很老土，但是很有效。
以下代码很直观的反应了这一点：

```c++
    // start the arg table in Lua
    lua_createtable(lua_state, 2, 0);
    lua_pushnumber(lua_state, 1);
    lua_pushnumber(lua_state, 49);
    lua_settable(lua_state, -3);
    lua_pushnumber(lua_state, 2);
    lua_pushstring(lua_state, "Life is a beach");
    lua_settable(lua_state, -3);
    lua_setglobal(lua_state, "arg");
```

这段代码的作用是创建类型为table的全局变量arg作为Lua的输入参数。
此table使用连续存储，并含有连个元素：

```lua
arg[1] = 49
arg[2] = "Life is a beach"
```

这段代码可以分为三部分：



##### 创建table
第一部分通过lua_createtable()函数完成，此函数创建一个匿名table并把它存入堆栈的顶部，其定义如下：

```c++
void lua_createtable(lua_State *L, int narr, int nrec);
```

其中参数narr和nrec分别用于指明连续存储元素和非连续存储元素的的数量。因为我们的table只有两个连续存储的元素，所以我们传递的实参为2和0.

##### 填充table
第二部分，用期望的值填充table。首先，用lua_push*()往Lua堆栈压入两个值，第一个是table的下标，第二个是下标对应得值。然后，通过lua_settable(lua_state, -3)将上述下标和对应的值存入table，-3代表从堆栈顶部往下数第三个数据，亦即是我们创建的匿名table。
以上操作相当于如下代码：

```lua
t[k] = v
```

v,k分别是堆栈顶部第一和第二个元素。
因此，以下代码：

```c++
    lua_pushnumber(lua_state, 1);
    lua_pushnumber(lua_state, 49);
    lua_settable(lua_state, -3);
```

就是设置我们的匿名table的下标1对应的值为49。需要注意的以上操作也将下标和对应的值弹出了堆栈，此时匿名table变成栈顶元素。

##### 命名table
第三部分，为创建和填充的table起个名字。这一步可以用lua_setglobal()函数，将table命名为全局arg的同时也将其弹出堆栈。此后，Lua就可以在脚本中识别这个table。
一旦脚本运行，其返回值也可以通过Lua堆栈传递给c++。需要特别注意的是，Lua将返回值按return语句后的表达式，顺序入栈，逆序出栈。
因为附带着冗余的系统信息，这段操作返回的信息量很大。基本上分为四步：

用lua_gettop()函数获取栈顶的具体位置
用lua_type()函数获取栈中具体位置的数据类型
用lua_to*()函数将堆栈中的数据转换为c++对应的类型
用lua_pop()函数弹出栈中的数据

这些函数的名字很好的解释其作用。
就是这样，很简单！

此文讨论了宿主语言和Lua共享数据的基本概念。你可以修改Lua脚本用以观察程序的输出变化。务必记住，你不需要重新编译就可以改变其执行逻辑。
下篇文章将讨论如何在Lua脚本中调用c++函数，敬请期待。