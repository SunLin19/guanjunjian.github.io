---
layout:     post
title:      "「二十」《C++Primber》要点提炼"
date:       2018-01-19 15:00:00
author:     "guanjunjian"
categories: 阅读
tags:
    - study
    - summary
---

* content
{:toc}

>
> 《C++Primer》 （第5版）要点提炼
>

# 第1章 开始

## 1.1 编写一个简单的C++程序

### 1.1.1 编译、运行程序

#### 程序源文件命名约定

-   不同编译器使用不同的后缀，最常见的包括`.cc、.cxx、.cpp、.cp、.C`

#### 从命令行运行编译器

-   `$ CC prog1.cc`
-   Windows生成可执行文件prog1.exe；UNIX会生成a.out
-   在Windows运行一个可执行文件可以忽略扩展名`$ prog1`或`$ .\prog1`
-   在UNIX运行可执行文件，需要全名`$ a.out`或`$ ./a.out`
-   通过echo命令获取返回值
    -   UNIX：`$ echo $?`
    -   Windows:`$ echo %ERRORLEVEL%`
-   最常用的编译器是GNU编译器和微软Visual Studio
    -   GNU的命令是g++:`$ g++ -o prog1 prog1.cc`。`$`是系统提示符，`-o prog1`指定了可执行文件的文件名，生成一个名为prog1（UNIX）或prog1.exe（Windows）的可执行文件
    -   省略-o prog1，则a.out（UNIX）或a.exe（Windows）
    -   微软Visual Studio 2010:`C:\Users\me\Programs> c1 /EHsc prog1.cpp`。`/EHsc`用来打开标准异常处理。这里将生成一个可执行文件，其名字与第一个源文件名对应，后缀为.exe

---




## 1.2 初识输入输出

-   C++语言并未定义任何输入输出（IO）语句，由标准库来提供
-   iostream库，包含两个基础类型：
    -   istream：输入流
    -   ostream：输出流
-   一个流就是一个字符序列

#### 标准输入输出对象

-   4个IO对象
    -   cin：istream类型的对象，标准输入
    -   cout：ostream类型的对象，标准输出
    -   ceer：输出警告和错误消息
    -   clog：用来输出程序运行时一般性信息

#### 一个使用IO库的程序

```c++
#include <iostream>
int main()
{
    std::cout << "Enter two numbers:"<<std::endl;
    int v1 = 0, v2 = 0;
    std::cin >> v1 >> v2;
}
```