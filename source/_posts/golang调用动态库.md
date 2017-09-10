---
title: golang调用c动态库
date: 2017-09-10 17:07:16
tags:
  - go
  - c
categories:
  - go
---

golang以其高并发，部署简单，编译迅速，而且像动态语言一样的开发效率等特点，吸引了很多程序的关注，但是有时候为了运行速度需要跟c一起使用，现整理一个golang调用c动态链接库的demo程序，以供自己将来参考，以下文字，大多来源于网络，整理如下。

<!-- more -->

1. 创建项目

    ```text
    ../cgolearn
    ├── lib
    │   ├── test_so.c
    │   └── test_so.h
    └── src
        ├── load_so.c
        ├── load_so.h
        └── main.go
    ```

2. 创建动态连接库

    lib目录下文件内容如下:
    ```go
    //test_so.h

    int test_so_func(int a,int b);
    ```

    ```go
    //test_so.c

    #include "test_so.h"

    int test_so_func(int a,int b)
    {
        return a*b;
    }
    ```

    生成so文件

    ```bash
    cd lib
    gcc -shared ./test_so.c -o test_so.so
    ```

3. 在golang代码中调用c动态库函数
    src目录下文件内容如下:
    ```c
    //load_so.h

    int do_test_so_func(int a,int b);
    ```

    ```c
    //load_so.c

    #include "load_so.h"
    #include <dlfcn.h>

    int do_test_so_func(int a,int b)
    {
        void* handle;
        typedef int (*FPTR)(int,int);

        handle = dlopen("./test_so.so", 1);
        FPTR fptr = (FPTR)dlsym(handle, "test_so_func");

        int result = (*fptr)(a,b);
        return result;
    }
    ```

    ```go
    //main.go

    package main

    /*
    #include "load_so.h"
    #cgo LDFLAGS: -ldl
    */
    import "C"
    import "fmt"

    func main() {
        fmt.Println("20*30=", C.do_test_so_func(20, 30))
    }
    ```

    编译运行:

    ```bash
    cd src
    go build -o main
    ./main
    ```


4. 注意事项
    1，import "C"，一定要紧跟着cgo标志的注释代码，中间不能有空行。
    2，import "C"必须单独一行，不能和其它库一起导入。

5. 待做事项
    do_test_so_func可以优化成一个工厂函数，从Go代码中传入不同的标志，调用不同的函数，返回不同类型的返回值。