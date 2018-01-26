---
layout:     post
title:      "「二十一」《C++Primber》笔记 第Ⅱ部分 "
date:       2018-01-26 20:00:00
author:     "guanjunjian"
categories: 阅读
tags:
    - study
    - summary
---

* content
{:toc}

>
> 《C++Primer》笔记 第Ⅱ部分 C++标准库
> 
> 包括第8至12章
> 
> [第8章 IO库] [第9章 顺序容器] [第10章 泛型算法] [第11章 关联容器] [第12章 动态内存]
> 
> 第五版 
>

# 第8章 IO库

## 8.1 IO类

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_8_1.png)

-   从上表图可以看出，标准库定义了一组类型和对象来操作`wchar_t`类型的数据，宽字符版本的类型和函数的名字都以一个w开始

#### IO类型间的关系

-   设备类型和字符大小都不贵影响我们执行IO操作
-   标准库使我们能忽略这些不同类型的流之间的差异，这是通过继承机制实现的
-   类fstream和stringstream都是继承自类iostream的，输入类都继承自istream，输出类都继承自ostream，因此可以在istream对象上执行的操作，也可以在ifstream和istringstream对象上执行，继承自ostream的输出类也有类型情况（源自本章小结）




### 8.1.1 IO对象无拷贝或赋值

-   不能拷贝或对IO对象赋值
-   因此不能将形成或返回类型设置为流类型，通常以引用方式传递和返回流
-   读写一个IO对象会改变其状态，因此传递和返回的引用不能是const的

### 8.1.2 条件状态

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_8_2.png)

-   一个流一旦发生错误，其上后续的IO操作都会失败，只有当流都处于无错状态时，我们才可以从它读取数据，向它写入数据
-   使用一个流之前检查它是否处于良好状态`while(cin >> word)`

#### 查询流的状态

-   iostate类型听过了表达流状态的完整功能
-   IO库定义了4个iostate类型的constexpr值表示特定的位模式，这些值用来表示特定类型的IO条件，可以与位运算符一起使用来一次性检测或设置多个标志位
-   badbit表示系统错误，如不可恢复的读写错误
-   failbit表示可恢复错误，如希望读取数值却读出一个字符等错误
-   goodbit的值为0，表示未发生错误
-   eofbit表示遇到文件结束位置，同时也会将failbit置位（设为1）
-   badbit、failbit、eofbit任一个被置位，则检测流状态的条件会失败
-   使用good或fail是确定流的总体状态的正确方法

#### 管理条件状态

-   clear是一个重载的成员
    -   不接受参数的版本：清除（复位，全设为0）所有错误标志位
    -   接受一个iostate类型参数的版本：复位单一的条件状态

```c
//复位failbit和badbit，保持其他标志位不变
cin.clear(cin.rdstate() & ~ cin.failbit & ~cin.badbit);
```

### 8.1.3 管理输出缓冲

-   每个输出流都管理一个缓冲区，用来保存程序读写的数据
-   导致缓冲刷新（即数据真正写到输出设备或文件）的原因
    -   程序正常结束，作为main函数的return操作的一部分，缓冲刷新被执行。如果程序异常终止，输出缓冲区时不会被刷新的，数据可能停留在缓冲区中等待打印
    -   缓冲区满时，需要刷新缓冲
    -   使用endl、flush、ends。其中endl换行后刷新缓冲区；flush刷新缓冲区但不输出任何额外的字符；ends向缓冲区插入一个空字符后刷新缓冲区
    -   每个输出操作之后，可以用unitbuf设置流的内部状态，来清空缓冲区。默认情况下，cerr是设置unitbuf的
    -   一个输出流可能被关联到另一个流。在这种情况下，当读写被关联的流时，关联到的流的缓冲区会被刷新。例如，默认情况下，cin和cerr都关联到cout，因此读cin或写cerr都会导致cout的缓存区被刷新（被关联的流刷新）

#### unitbuf操作符

```c
//所有输出操作都会立即刷新缓冲区
cout << unitbuf;  
//回到正常的缓冲方式
cout << nounitbuf
```

#### 关联输入和输出流

-   当一个输入流被关联到一个输出流时，任何试图从输入流读取数据的操作都会先刷新到关联的输出流
-   标准库将cout和cin关联在一起
-   tie是流的成员函数，有两个重载版本
    -   不带参数版本：返回指向输出流的指针。如果本对象当前关联到一个输出流，返回指向这个流的指针，如果未关联到流，返回空指针
    -   带接收一个指向ostream指针的版本：将调用该函数的流关联到此ostream，`x.tie(&o)`将流x关联到输出流o
-   可以将istream或ostream关联到另一个ostream

```c
cin.tie(&cout);  //cin关联到cout，cout调用时，会刷新cout的缓冲区
//old_tie指向当前cin关联到的cin流，没有则会空
//将cin不再与任何流关联
ostream *old_tie = cin.tie(nullptr); 
cin.tie(old_tie); // 重建关联
```

-   每个流同时最多关联到一个流，但多个流可以同时关联到同一个ostream

---

## 8.2 文件输入输出

-   头文件fstream定义了三个类型来支持文件IO：
    -   ifstream从一个给定文件读取数据
    -   ofstream向一个给定文件写入数据
    -   fstream可以读写给定文件
-   可以使用`<<`、`>>`以及`getline`，除此之外，还支持表8.3的操作

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_8_3.png)

### 8.2.1 使用文件流对象

#### 成员函数open和close

-   如果我们定义了一个空文件流对象，可以随后调用open来将它与文件关联起来

```c
ifstream in(ifile); //构筑一个ifstream并打开指定文件

ofstream out;  //输出流未与任何文件相关联
out.open(ifile + ".copy");  //打开指定流

if(out)  //检查open是否成功
```

#### 自动构造和析构

-   当一个fstream对象被销毁时，close会自动被调用

### 8.2.2 文件模式

-   每个流都有一个关联的文件模式

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_8_4.png)

-   无论用哪种方式打开文件（open打开或文件名初始化流fstream in fstream(file)），我们都可以指定模式
-   指定文件模式有如下限制
    -   只可以对ofstream或fstream对象设定out模式   
    -   只可以对ifstream或fstream对象设定in模式 
    -   只有当out也被设定时才可设置trunc模式
    -   只要trunc没被设定，就可以设定app模式。在app模式下，即是没有显示指定out模式，文件也总是以输出方式打开
    -   默认情况下，没有指定trunc，以out打开的文件也会被截断。为了保留out模式下打开的文件的内容，我们必须同时指定app，这样只会将数据追加到文件末尾；或者in模式，即对文件进行对鞋操作
    -   ate和binary可以用于任何类型的文件流对象，且可以与其他任何文件模式组合使用
-   默认情况：
    -   ifstream:in
    -   ofstream:out
    -   fstream:in out

#### 以out模式打开文件会丢弃已有数据

-   当我们打开一个ofstream时，文件的内容会被丢弃，阻止ofstream清空指定给定文件的方法是同时指定app（in也可以）：

```c
//在这几条语句中，file1都被截断
ofstream out("file1"); //隐含以out、trunc模式打开
ofstream out2("file1", ofstream::out); //隐含trunc打开
ofstream out3("file1", ofstream::out | ofstream::trunc);

//为了保留文件内容，必须显示指定app模式
ofstream app("file2", ofstream::app); //隐式out，效果与下句相同
ofstream app2("file2", ofstream::out | ofstream::app);
```

---

## 8.3 string流

-   头文件stringstream定义了三个类型来支持string IO：
    -   istringstream从一个给定string读取数据
    -   ostringstream向一个给定string写入数据
    -   stringstream可以读写给定文件

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_8_5.png)

```c
getline(cin,line);
//将绑定到刚刚读入的行
istringstream record(line);
```

