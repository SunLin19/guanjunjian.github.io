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
> 《C++Primer》 （第5版）要点提炼，持续更新
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
    std::cout << "Enter two numbers:" << std::endl;
    int v1 = 0, v2 = 0;
    std::cin >> v1 >> v2;
}
```

#### 向流写入数据

-   输出运算符（`<<`），`std::cout << "Enter two numbers:" << std::endl;`
-   `<<`接受两个运算对象：
    -   左侧：ostream对象
    -   右侧：要打印的值
-   若使用两次<<运算，因为此运算符返回其左侧的运算对象，所以第一个运算符的结果成为了第二个运算符的左侧运算对象，如下写法与上文中的效果一样
    -   `(std::cout << "Enter two numbers:") << std::endl;`
    -   `std::cout << "Enter two numbers:";    std::cout << std::endl`
-   endl是一个操作符的特殊值，效果是结束当前行，并将与设备关联的缓冲区中的内容刷到设备中。调试语句应该保持“一直”刷新流

#### 使用标准库中的名字

-   `std::`指出名字cout和endl是定义在名为std的命名空间中的。命名空间可以帮助我们避免不经意的名字冲突，以及使用库中相同名字导致的冲突
-   标准库在命名空间std中
-   作用域运算符（::）来指出我们使用定义在命名空间std中的名字cout

#### 从流读取数据

```c++
//以下三段句子效果一样
std::cin >> v1 >> v2;
( std::cin >> v1 ) >> v2;

std:: >> v1;
std:: >> v2;
```

-   输入运算符（>>）接受一个istream作为其左侧运算对象，接受一个对象作为其右侧运算对象，>>从给定的istream读入数据，并存入给定对象中
-   输入运算符返回其左侧运算对象作为其计算结果，即istream本身

---

## 1.4 控制流

### 1.4.3 读取数量不定的输入数据

```c++
while(std::cin >> value )
    statement;
```

-   此循环条件实际检测的是std::cin，因为std::cin >> value将值存在value之后，返回的是std::cin
-   当使用一个istream对象作为条件时，其效果是检测流的状态
    -   如果流是有效的，即未遇到错误，那么检测成功，即为真
    -   当遇到文件结束符、无效输入（即读入的值与>>右侧对象类型不一样），istream对象的状态会变为无效，处于无效时，itsream对象会使条件变为假
-   编译器可以检查出的错误：
    -   语法错误
    -   类型错误
    -   声明错误：每个名字都要先声明后使用，常见的有种错误：对来自标准库的名字忘记使用std::、标识符名字拼写错误

---

## 1.5 类简介

-   使用.h作为头文件后缀，也有用.H、.hpp或.hxx。标准库头文件通常不带后缀。大部分编译器不关心头文件名的形式
-   包含来自标准库的头文件，应该使用`<>`；不属于标准库的头文件，使用`""`
-   文件重定向`$ addItems < infile    > outfile`，其中addItems为可执行文件，该命令会从一个名为infile的文件读取数据，并将输出结果写到outfile

### 1.5.2 初识成员函数

-   成员函数是定义为类的一部分的函数，也称为方法
-   `item1.isbn()`使用`.`来表达“名为item1的对象的isbn成员”，点运算符只能用于类类型的对象，左侧为类类型的对象，右侧为该类型的成员名

## 小结

-   main函数是操作系统执行你的程序的调用入口