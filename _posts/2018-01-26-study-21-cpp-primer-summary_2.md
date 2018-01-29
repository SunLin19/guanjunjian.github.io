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

---

# 第9章 顺序容器

-   关于本章，可以参考[《c++顺序容器》(http://blog.csdn.net/wmz545546/article/details/77750878)]

-   一个容器就是一个特定类型对象的集合。
-   顺序容器为程序员提供了控制元素存储和访问顺序的能力，这种顺序不依赖于元素的值，与元素加入容器时的位置相对应
-   在11章将介绍有序和无序关联容器，则根据关联字的值来存储元素

---

## 9.1 顺序容器概述

-   这些容器在以下方面都有不同的性能折中
    -   向容器添加或从容器删除元素的代价
    -   非顺序访问容器中元素的代价

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_1.png)

-   string和vector将元素保存在连续的内存空间中。由于元素是连续存储的，由元素的下标计算其地址是非常快速的。但是，在这两种容器的中间位置添加或删除元素就会非常耗时：在第一次插入或删除操作之后，需要移动插入/删除位置之后的所有元素来保持连续存储。而且，添加一个元素有时还需要分配额外的存储空间，在这种情况下，每个元素都必须移动到新的存储空间中
-   list和`forward_list`的设计目的是令容器任何位置添加和删除操作都很快速，代价是，这两个容器不支持元素的随机访问：为了访问一个元素，需要遍历整个容器。与vector、deque和array相比，额外的内存开销也很大
-   deque支持快速的随机访问，中间位置添加或删除元素的代价代价（可能）很高，两端添加或删除都是很快的
-   `forward_list`和`array`是新C++标准添加的类型。
    -   与内置数组相比，array是一种更安全、更容易使用的数组类型，大小固定，不支持添加或删除元素以及改变容器大小的操作
    -   `forward_list`的设计目的是到达最好的手写的单向链表数据结构相当的性能，没有size操作，因为保持或计算其大小就会比手写链表多出额外的开销

---

## 9.2 容器库概览

-   9.2节将介绍对所有容器都适用的操作

-   每个容器都定义在一个头文件中，文件名与类型名相同
-   容器均定义为模板类，必须提供额外信息来生产特定的容器类型，例如`list<Sales_data>`

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_2_1.png)
![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_2_2.png)

### 9.2.1 迭代器

-   迭代器范围中的元素范围是一种左闭合区间，即`[begin,end)`，即end是一个尾后迭代器，迭代器范围中的元素包含begin至end（但不包含end）

#### 使用左闭合范围蕴含的编程假设

-   使用左闭合范围是因为这种范围有三种方便的性质
    -   如果begin和end相等，则范围为空
    -   如果begin和end不等，则范围至少有一个元素，且begin所指向该范围中的第一个元素
    -   我们可以对begin递增若干次，使得begin==end

### 9.2.2 容器类型成员

-   反向迭代器各种操作的含义都发送了颠倒。例如，对一个方向迭代器执行++，会得到上一个元素
-   通过类型别名，我们可以在不了解容器中元素的情况下使用它。
    -   如果需要元素类型，使用value_type
    -   需要元素类型的一个引用，使用reference或const_reference
    -   将在16章介绍
-   目前来说，为了使用这些类型，我们必须显示使用其类名：
```c
list<string>::iterator iter;
vector<int>::difference_type count;
```

### 9.2.3 begin和end成员

-   begin和end有多个版本
    -   带r的版本返回反向迭代器
    -   带c的版本返回const迭代器
-   实际上有两个名为begin的成员，rbegin、end和rend的情况类似
    -   一个是const成员，返回容器的const_iterator类型
    -   另一个是非常量成员，返回容器的iterator类型

### 9.2.4 容器定义和初始化

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_3.png)

#### 将一个容器初始化为另一个容器的拷贝

-   `C c1(c2)`和`C c1=c2`
-   当将一个容器初始化为另一个容器的拷贝时，两个容器的容器类型和元素类型都必须相同

#### 标准库array具有固定大小

-   标准库array的大小也是类型的一部分，当定义一个array时，除了指定元素类型，还要指定容器大小

```c
array<int,42>
array<string,10>

//使用array类型，也需要指定元素类型和大小
array<int,10>::size_type i;
array<int>::size_type j; //错误
```

-   默认构造的array是非空的：它包含了与其大小一样多的元素。这些元素都被默认初始化
-   对array进行列表初始化，初始值的数目必须等于或小于array的大小

```c
array<int,10> ia1;  //10个默认初始化的int
array<int,10> ia2 = {0,1,2,3,4,5,6,7,8,9}; //列表初始化
array<int,10> ia3 = {42}; //ia3[0]为42，剩余元素为0
```

-   不能对内置数组类型进行拷贝或对象赋值操作，但对array并无此限制

```c
int digs[10] = {0,1,2,3,4,5,6,7,8,9};
int cpy[10] = digs; //错误
array<int,10> digits = {0,1,2,3,4,5,6,7,8,9}
array<int,10> copy = digits; //正确，只要数组类型（元素类型和长度）匹配即合法
```

### 9.2.5 赋值和swap

-   赋值运算将运算符其左边容器中的全部元素替换为右边容器中元素的拷贝

```c
c1 = c2; //将c1的内容替换为c2中元素的拷贝
c1 = {a,b,c}; //赋值后c1的大小为3
```

-   第一个赋值运算后，左边容器将与右边容器相等。如果原来两个容器大小不同，则赋值运算后两者的大小都与右边容器的原大小相同
-   与内置数组不同，标准库array类型允许赋值。赋值号左右两边的运算对比必须具有相同的类型

```c
array<int,10> a1 = {1,2,3,4,5,6,7,8,9};
array<int,10> a2 = {0}; //所有元素均为0，在初始化的时候可以使用花括号和遗漏元素，参考表9.3
a1 = a2; //替换a1中的元素
a2 = {0}; //错误，不能将一个花括号列表赋予数组，参考表9.4 
```

-   由于右边运算对象的大小可能与左边运算对象的大小不同，因此array类型不支持assign，也不允许用花括号包围的值列表进行赋值

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_4.png)

#### 使用assign(仅顺序容器)

-   顺序容器（array除外）还定义了一个名为assign的成员，允许我们从一个不同但**相容**的类型赋值，或者从容器的一个**子序列**赋值。assign操作用参数指定的元素（的拷贝）替换左边容器中的所有元素。例如，我们可以用assign实现将一个vector中的一段char*赋值予一个list中的string

```c
list<string> names;
vector<const char*> oldstyle;
names = oldstyle;   // 错误：容器类型不匹配
// 正确：可以将const char*转换为string
name.assign(oldstyle.cbegin(), oldstyle.cend());
```

-   由于其旧元素被替换，因此传递给assign的迭代器不能指向调用assign的容器

#### 使用swap

-   除array外，swap不对任何元素进行拷贝、删除或插入操作，因此可以保证在常数时间内完成
-   元素不会被移动的事实意味着，除string外，指向容器的迭代器、引用和指针在swap操作之后都不会失效，它们仍然指向swap操作之前所指向的那些元素。但是，在swap之后，这些元素已经属于不同的容器了。例如，将定iter在swap之前指向svec1[3]的string，那么在swap之后它指向svec3[3]的元素。与其他容器不同，对一个string调用swap会导致迭代器、引用和指针失效。 
-   与其他容器不同，swap两个array会真正交换它们的元素，因此交换两个array所需的时间与array中元素的数目成正比。对于array，在swap操作之后，指针、引用和迭代器所绑定的元素保持不变，但元素值已经与另一个array中对应元素的值进行了交换。

### 9.2.7 关系运算符

-   关系运算符左右两边的运算对象必须是相同类型的容器，且必须保存相同类型的元素
-   这些运算符的工作方式与string的关系运算类
    -   如果两个容器具有相同大小且所有元素都两两对应相等，则这两个容器相等，否则两个容器不等
    -   如果两个容器大小不同，但较小容器中每个元素都等于较大容器中的对应元素，则较小容器小于较大容器
    -   如果两个容器都不是另一个容器的前缀子序列，则它们的比较结果取决于第一个不相等的元素的比较结果
-   只有当其元素类型也定义了相应的比较运算符时，我们才可以使用关系运算符来比较两个容器

---

## 9.3 顺序容器

-   本节介绍顺序容器所特有的操作

### 9.3.1 向顺序容器添加元素

-   除array外，所有标准库容器都提供灵活用的内容管理

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_5.png)

#### 使用push_back

-   当我们用一个对象来初始化容器时，或将一个对象插入到容器中时，实际上放入到容器中的时对象值的一个拷贝，而不是对象本身

#### 插入范围内元素

```c
//运行时错误：迭代器表示要拷贝的范围，不能指向与目的位置相同的容器，因为迭代器失效的问题
slist.insert(slist.begin(), slist.begin(), slist.end());
```

#### 使用emplace操作

-   当调用push或insert成员函数时，我们将元素类型的对象传递给它们，这些对象被拷贝 到容器中。而当我们调用一个emplace成员函数时，则是将参数传递给元素类型的构造函数，emplace成员使用这些参数在**容器管理的内存**中直接构造元素

```c
// 在c的末尾构造一个Sales_data对象
// 使用三个参数的Sales_data构造函数
c.emplace_back("9999-99999", 25, 15.99);

// 错误：没有接受三个参数的push_back版本
c.push_back("9999-99999", 25, 15.99);

// 正确：创建一个临时的Sales_data对象传递给push_back，等价于第1句
c.push_back(Sales_data("9999-99999", 25, 15.99);
```

-   调用`emplace_back`时，会在容器管理的内存空间中直接创建对象，而调用`push_back`则会创建一个局部临时对象，并将其压入容器中
-   emplace函数在容器中直接构造元素，传递给emplace函数的参数必须与元素类型的构造函数相匹配

```c
// iter 指向c 中一个元素，其中保存了Sa1es_data 元素
c.emp1ace_back(); //使用Sales data 的默认构造函数
c.emplace(iter, " 999- 999999999 " ); // 使用Sa1es_data(string)
//使用Sales_data 的接受一个ISBN 、一个count 和一个price 的构造函数
c.emplace_front(" 978 - 0590353403 " , 25 , 15.99);
```

### 9.3.2 访问元素

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_6.png)

#### 访问成员函数返回的是引用

-   在容器中访问元素的成员函数（即，front、back、下标和at），返回的元素都是引用

```c
if(!c.empty()){
    c.front() = 42;
    auto &v = c.back();
    v = 1024;  //改变c中的元素
    auto v2 = c.back; //这里v2不是一个引用
    v2 = 0; //未改变c中的元素
}
```

-   如果我们使用auto变量来保存这些函数的返回值，并且希望使用此变量来改变元素的值，必须记得将变量定义为引用类型

### 9.3.3 删除元素

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_7.png)

-   删除元素的成员函数并不检查其参数，在删除元素之前，程序员必须确保它（们）是存在的

#### 删除对个元素

```c
// 删除两个迭代器表示的范围内的元素
// 返回指向最后一个被删元素之后位置的迭代器
elem1 = slist.erase(elem1, elem2); // 调用后，elem1 == elem2
```

### 9.3.4 特殊的forward_list操作

-   在一个`forward_list`中添加或删除元素的操作是通过改变给定元素之后的元素来完成的，这样我们总是可以访问到被添加或删除操作所影响的元素。由于这些操作与其他容器上的操作的实现方式不同，`forward_list`并未定义insert、emplace和erase，而是定义了名为`insert_after`、`emplace_after`和`erase_after`的操作

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_8.png)

### 改变容器大小

-   我们可以用resize 来增大或缩小容器，array不支持resize
-   如果当前大小大于所要求的大小，容器后部地元素会被删除；如果当前大小小于新大小，会将新元素添加到容器后部

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_9.png)

### 9.3.6 容器操作可能使迭代器失效

-   向容器中添加元素和从容器中删除元素的操作可能会是指向容器元素的指针、引用或迭代器失效
-   在向容器添加元素后：
    -   如果容器是vector或string，且存储空间被重新分配，则指向容器的迭代器、指针和引用都会失败。如果存储空间未重新分配，指向插入位置之前的元素的迭代器、指针和引用仍有效，但指向插入位置之后元素的迭代器、指针和引用将会失效
    -   对于deque，插入到除首尾位置之外的任何位置都会导致迭代器、指针和引用失效。如果在首尾位置添加元素，迭代器会失效，但指向存在的元素的引用和指针不会失效
    -   对于list和forward_list，指向容器的迭代器（包括尾后迭代器 和首前迭代器 ）、指针和引用仍有效
-   当我们删除一个元素后：
    -   对于vector和string，指向被删除元素之前元素的迭代器、引用和指针仍有效。注意，当我们删除元素时，尾后迭代器总是会失效
    -   对于deque，如果在首尾之外的任何位置删除元素，那么指向被删除元素外其他元素的迭代器、引用或指针也会失效。如果是删除deque的尾元素，则尾后迭代器 也会失效，但其他迭代器、引用和指针不受影响；如果是删除首元素，这些也不会受影响
    -   对于list和 forward_list ，指向容器其他位置的迭代器（包括尾后迭代器 和首前迭代器 ）、引用和指针仍有效
-   使用失效的迭代器、指针或引用是严重的运行时错误
-   由于向迭代器添加元素和从迭代器删除元素的代码可能会使迭代器失效，因此必须保证每次改变容器的操作之后都正确地重新定位迭代器，这个建议对vector 、string 和deque尤为重要

```c
auto begin = v.begin(), end = v.end();
while(begin != end)  //end有可能失效

while(begin != v.end)  //每次都能获得最新的end
```

-   如果在一个循环中插入/删除deque、string或vector中的元素，不要缓存end返回的迭代器

---

## vector对象是如何增长的

-   为了支持快速随机访问，vector将元素连续存储，每个元素紧挨着前一个元素存储
-   如果没有空间容纳新元素，容器不可能简单地将它添加到内存中其他位置——因为元素必须连续储存。容器必须分配新的内存空间来保存已有元素和新元素，将已有元素从旧元素从久位置移动到新空间中，然后添加新元素，释放存储空间
-   标准库实现者采用来可以减少容器空间重新分配次数的策略。当不得不获取新的内存空间时，vector和string的实现通常会分配比新的空间需求更大的内存空间（即预留空间）

#### 管理容器的成员函数

-   vector和sting类型提供来一些成员函数，允许我们与它的实现中内存分配部分互动

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_10.png)

-   c.reserve()
    -   reserve并不改变容器中元素的数量，它仅影响vector预先分配多大的内存空间
    -   只有当内存空间超过当前容量时，reserve调用才会改变vector的容量，如果需求大小大于当前容量，reserve至少分配与需求一样大的内存空间（可能更大）
    -   如果需求大小小于或等于当前容量，reserve什么也不做。特别是，当需求大小小于当前容量时，容器不会退回内存空间
    -   在调用reserve之后，capacity将会大于或等于传递给reserve的参数
-   resize成员函数 只改变容器中元素的数目，而不是容器的容量。我们同样不能用resize来减少容器预留的内存空间
-   我们可以调用`shrink_to_fit`来要求deque、vector或string退回不需要的内存空间。此函数指出我们不再需要任何多余的内存空间。但是，具体的实现可以选择忽略此请求，也就是说，调用`shrink_to_fit`也并不保证一定退回内存空间

#### capactiy和size

-   容器的size是指它已经保存的元素的数目；而capacity则是在不分配新的内存空间的前提下它最多可以保存多少元素（包括已经保存的元素）
-   只要没有操作需求超出vector的容量，vector就不能重新分配内存空间
-   每个vector实现都可以选择自己的内存分配策略，但必须遵守的一条原则是：只有迫不得已时才可以分配新的内存空间。只有在指向insert操作时size与capacity相等，或者调用resize或reserve时给定的大小超过当前capacity，vector才可能重新分配内存空间。分配多少超过给定容量的额外空间，取决于具体实现

---

## 9.5 额外的string操作

### 9.5.1 构造string的其他方法：

-   除了3.2.1和表9.3的构造方法外，string还支持一下三个构造函数：

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_11.png)

```c
char noNu l l [] = ( 'H', 'i' );   // 不是以空字符结束
string s3 (noNu11);  // 未定义noNu11 不是以空字符结束
```

-   通常当我们从一个const char*创建string时，指针指向的数组必须以空字符结尾，拷贝操作遇到空字符时停止。如果我们还传递给构造函数一个计数值，数组就不必以空字符结尾

#### substr操作

-   substr操作返回一个string，它时原始string的一部分或全部的拷贝，可以传递给substr一个可选的开始位置和计数值

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_12.png)

### 9.5.2 改变string的其他方法

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_13_1.png)
![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_13_2.png)

### 9.5.3 string搜索操作

-   每个搜索操作都会返回一个`string::size_type`值（unsigned类型），表示匹配发生位置的下标，如果搜索失败，则返回一个名为string::npos 的static成员。标准库将npos定义为一个const string::size_type类型，并初始化为值-1
-   由于npos是一个unsigned类型，此初始值意味着npos等于任何string最大的可能大小（参见2.1.2）

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_14_1.png)
![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_14_2.png)

### 9.5.4 compare函数

-   除了关系运算符外，标准库string类型还提供了一组compare函数，与C标准库的strcmp函数很相似

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_15.png)

### 9.5.5 数值转换

-   数值15如果保存为16位的short类型，则二进制位模式为0000000000001111
-   字符串"15"存在两个Latin-1编码的char，二进制位模式为0011000100110101
-   string参数中第一个非空白符必须是
    -   符号（+或-）或数字
    -   也可以以0x或0X开头表示十六进制
    -   对那些将字符串转换为浮点值的函数，也可以以小数点（.）开头，并可以包含E或e表示指数部分
    -   对于那些将字符串转换为整型值的函数，根据基数不同，string参数可以包含字母符，对应大于数字9的数
-   如果string不能转换为一个数值，抛出：invalid_argument异常
-   如果转换得到的数值无法用任何类型表示，抛出：`out_of_range`异常

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_16.png)

---

## 9.6 容器适配器

-   标准库定义来三个顺序容器适配器：stack 、queue 和priority_queue
-   适配器 是标准库中的一个通用概念。容器、迭代器和函数都有适配器
-   一个适配器是一种机制，能使某种事物的行为看起来像另外一种事物一样
-   一个容器适配器接受一个已有的容器类型，使其行为看起来像一种不同的类型
-   例如，stack适配器接受一个顺序容器（除array或forward_list外 ），并使其操作起来像一个stack一样
-   所有容器适配器都支持的操作和类型如下：

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_17.png)

#### 定义一个适配器

-   每个适配器都定义两个构造函数：
    -   默认构造函数创建一个空对象
    -   接收一个容器的构造函数拷贝该容器来初始化适配器\

```c
//从deq拷贝元素到stk
stack<unt> stk(deq);
```

-   默认情况下，stack和queue是基于deque实现的，`priority_queue`是在vector之上实现的
-   我们可以创建一个适配器时将一个命名的顺序容器作为第二个类型参数，来重载默认容器类型（即改变适配器的默认实现方式）

```
//改变了stack的实现方式，在vector上实现，这里声明了一个空栈
stack<string,vector<string>> str_stk;
//str_stk2也是在vector上实现，初始化时保存svec的拷贝
stack<string,vector<string>> str_stk2(svec);
```

-    对于一个给定的适配器，可以使用哪些容器是有限制的。所有适配器都要求容器具有添加和删除元素的能力 。因此，适配器不能构造在array之上
-    不能用forward_list来构造适配器，因为所有适配器都要求容器具有添加、删除以及访问尾元素的能力
-    三类适配器可以基于哪些容器实现：
    -   stack只要求`push_back`、`pop_back`和back操作，因此可以使用除array和forward_list 之外的任何容器类型来构造stack
    -   queue适配器要求back、`push_back`、front和`push_front`。因此它可以构造于list或deque之上， 但不能基于vector构造
    -   `priority_queue`除了front，`push_back`和`pop_back`操作之外还要求随机访问能力，因此它可以构造于vector或deque 之上，但不能基于list构造

#### 栈适配器

-   stack类型定义在stack头文件中

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_18.png)

-   每个容器适配器都基于底层容器类型的操作定义了自己的特殊操作。我们只可以使用适配器操作，而不能使用底层容器类型的操作

#### 队列适配器

-   queue和`priority_queue`适配器定义在queue头文件中

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_19_1.png)
![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_9_19_2.png)

-   标准库queue使用一种先进先出的存储和访问策略
-   `priority_queue`允许我们为队列中的元素建立优先级。新加入的元素会排在所有优先级比它低的已有元素之前。标准库在元素类型上使用`<`运算符来确定相对优先级。将在11.2.2学习如何充值这个默认设置

---

# 第10章 泛型算法

-   这些算法中的大多数都独立于任何指定的容器
-   这些算法是通用的：它们可用于不同类型的容器和不同类型的元素，还能用于其他类型的序列

---

## 10.1 概述

-   大多数算法在头文件`algorithm`，一组数值泛型算法还在头文件`numeric`
-   一般情况下，这些算法并不直接操作容器，而是遍历由两个迭代器指定的一个元素范围来进行操作，对其中每个元素进行一些处理
-   如find函数，它的工作是在一个未排序的元素序列中查找一个特定元素：

```c
int val = 42;
auto result = find(vec.cbegin(),vec.cend(),val);
//返回指向第一个等于给定值的元素的迭代器
//如果范围中无匹配元素，则find返回第二个参数来表示搜索失败
//可以通过比较返回值和第二个参数来判断是否成功
```

#### 迭代器令算法不依赖于容器，……

-   用迭代器操作来实现
-   如果发现匹配元素，find可以返回指向该元素的迭代器
-   find可以返回尾后迭代器来表示未找到给定元素

#### ……，但算法依赖于元素类型的操作

-   大多数算法都适用了一个（或多个）元素类型上的操作、
-   如find使用了==运算符
-   其他算法可能还要求元素类型支持<运算符

#### 算法永远不会执行容器的操作

-   算法永远不会改变底层容器的大小
-   算法可能改变容器中保存的元素的值，也可能在容器内移动元素，但永远不会直接添加或删除元素

---

## 10.2 初识泛型算法

### 10.2.1 只读算法

-   一些算法只会读取其范围内的元素，而从不改变元素，如find、accumulate
-   accumulate定义在numeric中

#### 算法和元素类型

```c
//对vec中的元素求和，和的初始值是0
//第3个参数决定使用哪个类型的加法运算以及返回类型
//序列中的元素类型必须与第3个参数匹配或能转换成第3个参数的类型
//这里的0表示使用的是int的+
int sum = accumulate(vec.cbegin(),vec.cend(),0);
//这里的string("")表示使用的是string的+，即string元素连接
string sum = accumulate(v.cbegin(),v.cend(),string(""));
//错误，const char*上没有定义+运算符
string sum = accumulate(v.cbegin(),v.cend(),"");
```

#### 操作两个序列的算法

-   equal用于确定两个序列是否保存相同的值

```c
//rostre2中的元素数目至少与rostre1一样多
equal(rostre1.cbegin(),rostre1.cend(),rostre2.cbegin())
```

-   可以通过equal来比较两个不同类型的容器中的元素。元素类型也不必一样，只要能用==来比较两个元素即可
-   那些只接受一个单一迭代器来表示第二个序列的算法，都假定第二个序列至少与第一个序列一样长

### 10.2.2 写容器元素的算法

-   `fill`函数

```c
fill(vec.begin(),vec.end(),0); //将每个元素重置为0
fill(vec.begin(),vec.begin()+vec.size()/2,10); //将容器的一个子序列设置为0
```

#### 算法不检查写操作

-   `fill_n`接受一个单迭代器、一个计数器和一个值
-   `fill_n(dst,n,val)` dst指向一个元素，而从dst开始的序列至少包含n个元素，该函数把这n个值设为val

```c
vector<int> vec; //空vector
//从首元素开始，将size个元素重置为0，由于这里vector为空，那么size为0，该语句正确
fill_n(vec.begin(),vec.size(),0);
//该语句错误，因为vec是个空向量，vec中的10个元素不存在
fill_n(vec.begin(),10,0)
```

#### 介绍back_inserter

-   一种确保算法有足够的元素空间存储输出数据的方法是使用插入迭代器
-   插入迭代器是一种向容器中添加元素的迭代器
-   当我们通过一个**（普通）迭代器**向容器元素赋值时，值被赋予迭代器指向的元素
-   通过一个**插入迭代器**赋值时，一个与赋值号右侧相等的元素被添加到容器中
-   `back_inserter`，定义在iterator头文件
-   `back_inserter`接受一个指向容器的引用，返回一个与该容器绑定的插入迭代器

```c
vector<int> vec; //空vector
auto it = back_inserter(vec); //通过它赋值会将元素添加到vec中
*it = 42; //vec现在有一个元素，值为42

vector<int> vec1; //空vector
//正确，创建的插入迭代器可以用来向vec添加元素，每次赋值都会在vec上调用push_back
fill_n(back_inserter(vec1),10,0)
```

#### 拷贝算法

-   拷贝算法是另一个向目的位置迭代器指向的输出序列中的元素写入数据的算法
-   接收三个迭代器，前两个表示输入范围，第三个表示目睹序列的起始位置

```c
int a1[] = {0,1,2,3,4,5,6,7,8,9};
int a [sizeof(a1)/sizeof(*a1)];
//ret指向拷贝到a2的尾元素之后的位置即a2[10]
auto ret = copy(begin(a1),end(a1),a2); 
```

-   replace算法
-   4个参数：前两个迭代器，表示输入序列，后两个一个是要搜索的值，另一个是新值

```c
//将所有值为0的元素改为42
replace(ilist.begin(),ilist.end(),0,42);
```

-   如果希望保留原序列不变，可以调用`replace_copy`

```c
//ilist并未改变
//ivec包含ilist的一份拷贝，不过原来在ilist中值为0的元素在ivec中变为42
replace_copy(ilist.cbegin(),ilist.cend(),back_inserter(ivec),0,42);
```

### 10.2.3 重排容器元素的算法

-   sort会重排输入序列中的元素，利用元素类型的<运算符来实现

```c
//按字典排序words
sort(words.begin(),words.end());
```

-   unique算法重排输入序列，将相邻的重复项“消除”
-   unique并不真的删除任何元素，它是覆盖相邻的重复元素
-   unique返回的迭代器向最好一个不重复之后的位置。此位置之后的元素仍然存储，但我们不知道它们的值是什么

```
//unique重排输入范围，使得每个单词只出现一次
//排序在范围的前部，返回指向不重复区域之后一个位置的迭代器
auto end_unique = unique(words.begin,words.end());
```

---

## 10.3 定制操作

-   参考[《定制操作(传递函数或lambda表达式)》](https://www.cnblogs.com/wuchanming/p/3918395.html)

### 10.3.1 向算法传递函数

#### 谓词

-   谓词是一个可调用的表达式，其返回结果是一个能用作条件的值
-   标准库算法所使用的谓词分为两类：一元谓词（意味着它们只接受单一参数）和二元谓词（意味着它们接受两个参数）
-   接受谓词参数的算法对输入序列中的元素调用谓词。因此，元素类型必须能转换为谓词的参数类型

```c
//接受一个二元谓词参数的sort版本用这个谓词代替<来比较元素
//比较函数，用来比较长度排序单词
bool isShorter(const string &s1,const string &s2)
{
　　return s1.size()<s2.size();
}
//按长度由短至长排序words
sort(words.begin(),words.end(),isShorter);
```

#### 排序算法

-   在我们将words按大小重排的同时，还希望具有相同长度的元素按字典序排列。为了保持相同长度的单词按字典序排列，可以使用stable_sort算法。这种稳定排序算法维持相等元素的原有顺序
-   通过调用stable_sort，可以保持等长元素间的字典序

```c
elimDups(words);//将words按字典序重排，并消除重复单词
//按长度重新排序，长度相同的单词维持字典序
stable_sort(words.begin(),words.end(),isShorter);
for(const auto &s:words)
　　cout<<s<<" ";
cout<<endl;
```

### 10.3.2 lambda表达式

-   我们可以使用标准库find_if算法来查找第一个具有特定大小的元素
-   find_if算法接受一对迭代器，表示一个范围
-   `find_if`的第三个参数是一个谓词。`find_if`算法对输入序列中的每个元素调用给定的这个谓词
-   它返回第一个使谓词返回非0值的元素，如果不存在这样的元素，则返回尾迭代器

#### 介绍lambda

-   可调用对象：对于一个对象或一个表达式，如果可以对其使用调用运算符，则称它为可调用的
-   可调用对象有：
    -   函数
    -   函数指针
    -   重载了函数调用运算符的类
    -   lambda表达式
-   一个lambda表达式表示一个可调用的代码单元。我们可以将其理解为一个未命名的内联函数。一个lambda具有一个返回返回类型、一个参数列表和一个函数体。lambda可能定义在函数内部
-   一个lambda表达式具有如下形式：

```c
/* 
@ [capture list]:捕获列表，是一个lambda所在函数中定义的局部变量列表（通常为空）
@ return type、parameter list和function body与任何普通函数一样，分别表示返回类型、参数列表和函数体
@ lambda必须使用尾置返回来指定返回类型
*/
[capture list] (parameter list) ->return type { function body }
```

```c
//可以忽略参数列表和返回类型，但必须永远包含捕获列表和函数体
auto f=[] {return 42;};

//lambda的调用方式
cout<<f()<<endl; //打印42
```

-   在lambda中忽略括号和参数列表等价于指定一个空参数列表
-   忽略返回类型，则返回类型从返回的表达式的类型推断而来
    -   如果函数体只有一个return语句，则返回类型从返回的表达式的代码推断出返回类型   
    -   如果lambda的函数体包含任何单一return语句之外的语句，且未指定返回类型，则返回void

#### 向lambda传递参数

-   lambda不能有默认参数
-   一个带参数的lambda的例子：

```c
//一个与isShorter函数完成相同功能的lambda
//空捕获列表表明此lambda不使用它所在函数中的任何局部变量
[] (const string &s1,const string &s2)
    { return s1.size()<s2.size();}
    
//使用此lambda来调用stable_sort
//当stable_sort需要比较两个元素时，它就会调用给定的这个lambda表达式
stable_sort(words.begin(),words.end(),[](const string &s1,const string &s2)
　　　　　　　　　　　　　　　　　　{return s1.size()<s2.size();});
```

#### 使用捕获列表

-   一个lambda可以出现在一个函数中，使用其局部变量，但它只能使用那些明确指明的变量
-   一个lambda通过将局部变量包含在其捕获列表中指出将会使用这些变量
-   捕获列表指引lambda在其内部包含访问局部变量所需的信息
-   使用捕获列表的例子：
-   
```c
//ambda会捕获sz
[sz] (const string &s)
　　{return s.size()>=sz;};

//错误：sz未捕获
[] (const string &s);
　　{ return s.size()>=sz;};
```

-   一个lambda只有在其捕获列表中捕获一个所在函数中的局部变量，才能在函数体中使用该变量

#### 调用find_if

```c
auto wc=find_if(words.begin(),words.end(),
　　　　　　　[sz] (const string &s)
　　　　　　　　{return s.size()>=sz;});
```

#### for_each算法

-   for_each算法：此算法接受一个可调用对象，并对输入序列中每个元素调用此对象

```c
//打印长度大于等于给定值的单词，每个单词后面接一个空格
//其函数体中还是使用了两个名字：s和cout，前者是它自己的参数
//cout不是定义在biggies中的局部名字，而是定义在头文件iostream中。因此，只要在biggies出现的作用域中包含了头文件iostream，我们的lambda就可以使用cout
for_each(wc,words.end(),
　　[](const string &s) {cout<<s<<" ";});
cout<<endl;
```

### 10.3.3 lambda捕获和返回

-   当定义一个lambda时，编译器生成一个与lambda对应的新的（未命名的）类类型。将在14.8.1介绍如何生成这种类
-   当向一个函数传递一个lambda时，同时定义了一个新类型和该类型的一个对象：传递的参数就是此编译器生成的类类型的未命名对象
-   当使用auto定义一个用lambda初始化的变量时，定义了一个从lambda生成的类型的对象
-   默认情况下，从lambda生成的类都包含一个对应该lambda所捕获的变量的数据成员
-   lambda的数据成员在lambda对象创建时被初始化

#### 值捕获

-   变量的捕获方式也可以是值或引用
-   采用值捕获的前提是变量可以拷贝
-   被捕获的变量的值是在lambda创建时拷贝，而不是调用时拷贝

```c
void fcn1()
{
　　size_t v1=42;  //局部变量
　　//将v1拷贝到名为f的可调用对象
　　auto f=[v1] {return v1;};
　　v1=0;
　　auto j=f(); //j为42；f保存了我们创建它时v1的拷贝
}
```

#### 引用捕获

```c
void fcn2()
{
　　size_t v1=42;
　　//对象f2包含v1的引用
　　auto f2=[&v1] { return v1;};
　　v1=0;
　　auto j=f2();  //j为0；f2保存v1的引用，而非拷贝
}
```

-   如果我们采用引用方式捕获一个变量，就必须确保被引用的对象在lambda执行的时候是存在的
-   我们也可以从一个函数返回lambda，函数可以直接返回一个可调用对象，或者返回一个类对象，该类含有可调用对象的数据成员。如果函数返回一个lambda，则与函数不能返回一个局部变量的引用类似，此lambda也不能包含引用捕获。

#### 隐式捕获

-   让编译器根据lambda体中的代码来推断我们要使用哪些变量
-   在捕获列表中写一个&或=
    -   &告诉编译器采用捕获引用方式
    -   =表示采用值捕获方式
-   隐式捕获，值捕获方式的例子：

```c
wc=find_if(words.begin(),words.end(),
　　[=] (const string &s)
　　　　{ return s.size()>=sz; });
```

-   混合使用隐式捕获和显式捕获的例子：

```c
void biggies(vector<string> &words,vector<string>::size_type sz,ostream &os=cout,char c=' ')
{
　　//os隐式捕获，引用捕获方式；c显式捕获，值捕获方式
　　for_each(words.begin(),words.end(),[&,c](const string &s) {os<<s<<c;});
　　//os显式捕获，引用捕获方式；c隐式捕获，值捕获方式
　　for_each(words.begin(),words.end(),[=,&os](const string &s) {os<<s<<c;});
}
```

-   当我们混合使用隐式捕获和显式捕获时，捕获列表中的第一个元素必须是一个&或=，此符号指定了默认捕获方式为引用或值
-   当混合使用隐式捕获和显式捕获时，显式捕获的变量必须使用与隐式捕获不同的方式。即，如果隐式捕获是引用方式（使用了&），则显示捕获命名变量必须采用值方式，因此不能在其名字前使用&。如果隐式捕获采用的是值方式（使用了=），则显式捕获命名变量必须采用引用方式，即，在名字前使用&

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_10_1.png)

#### 可变lambda

-   默认情况下，对于一个值被拷贝的变量，lambda不会改变其值。如果我们希望能改变一个捕获的变量的值，就必须在参数列表首加上关键字mutable
-   可变lambda能省略参数列表

```c
void fcn3()
{
　　size_t v1=42; //局部变量
　　//f可以改变它所捕获的变量的值
　　auto f=[v1] () mutable {return ++v1;};
　　v1=0;
　　auto j=f(); //j为43
}
```

-   一个引用捕获的变量是否（如往常一样）可以修改，依赖于此引用指向的是一个const类型还是一个非const类型：

```c
void fcn4()
{
　　size_t v1=42;  //局部变量
　　//v1是一个非const变量的引用
　　//可以通过f2中的引用来改变它
　　auto f2=[&v1] {return ++v1;};
　　v1=0;
　　auto j=f2(); //j为1
}
```

#### 指定lambda返回类型

-   在不指定返回类型的情况下
    -   如果函数体只有一个return语句，则返回类型从返回的表达式的代码推断出返回类型   
    -   如果一个lambda体包含return之外的任何语句，则编译器假定此lambda返回void

```c
//正确，只有一个return语句，根据返回表达式推断返回类型
transform(vi.begin(),vi.end(),vi.begin(), [] (int i) {return i<0?-i:i;});

//错误，不能推断lambda的返回类型，则返回类型为void，这里返回了int是错误的
transform(vi.begin(),vi.end(),vi.begin(),[] (int i) {if(i<0) return -i; else return i;});
```

-   当我们需要为一个lambda定义返回类型时，必须使用尾置返回类型：

```c
transform(vi.begin(),vi.end(),vi.begin(),[] (int i)->int  {if(i<0) return -i; else return i;});
```

### 10.3.4

-   参考[《参数绑定》](http://www.cnblogs.com/wuchanming/p/3918403.html)
-   如果lambda的捕获列表为空，通常可以用函数来代替它
-   但是，对于捕获局部变量的lambda，用函数来替换它就不是那么容易了
-   例如，我们用在find_if调用中的lambda比较一个string和一个给定大小。我们可以很容易地编写一个完成同样工作的函数：

```c
bool check_size(const string &s,string::size_type sz)
{
　　return s.size()>=sz;
}
```

-   但是，我们不能用这个函数作为find_if的一个参数。
-   `find_if`接受一个一元谓词，因此传递给find_if的可调用对象必须接受单一参数
-   binggies传递给`find_if`的lambda使用捕获列表来保存sz。为了用`check_size`来代替此lambda，必须解决如何向sz形参传递一个参数的问题

#### 标准库bind函数

-   bind标准库函数，定义在头文件functional中，可以将bind函数看作一个通用的函数适配器，它接受一个可调用对象，生成一个新的可调用对象来“适应”原对象的参数列表
-   调用bind的一般形式：

```c
auto newCallable=bind(callable,arg_list);
```

-   当我们调用newCallable时，newCallable会调用callable，并传递给它arg_list中的参数
-   `arg_list`中的参数可能包含形如`_n`的名字，其中n是一个整数。这些参数是“占位符”，表示newCallable的参数。数值n表示生成的可调用对象中参数的位置，`_1`为newCallable的第一个参数，`_2`为第二个参数，以此类推

#### 绑定check_size的sz参数

```c
//check6是一个可调用对象，接受一个string类型的参数
//并用此string和值6来调用check_size
auto check6=bind(check_size,_1,6);
/*
此bind调用只有一个占位符，表示check6只接受单一参数。
占位符出现在arg_list的第一个位置，表示check6的此参赛对应check_size的第一个参数。此参数是一个const string&。
因此，调用check6必须传递给它一个string类型的参数，check6会将此参数传递给check_size。
*/

string s="hello";
bool b1=check6(s); //check6(s)会调用check_size(s,6)
```

-   使用bind，我们可以将原来基于lambda的`find_if`调用，替换为如下使用`check_size`的版本：

```c
//基于lambda的find_if
auto wc=find_if(words.begin(),words.end(),[sz](const string &s));
//替换为
auto wc=find_if(words.begin(),words.end(),bind(check_size,_1,sz));
```

-   此bind调用生成一个可调用对象，将check_size的第二个参数绑定到sz的值
-   当`find_if`对words中的string调用这个对象时，这些对象会调用`check_size`，将给定的string和sz传递给它
-   因此，`find_if`可以有效地对输入序列中每个string调用`check_size`，实现string的大小与sz的比较

#### 使用placeholders名字

-   名字_n都定义在一个名为placeholders的命名空间中，而这个命名空间本身定义在std命名空间中
-   placeholders命名空间定义在functional头文件中

```c
using std::placeholders::_1;
```

-   对每个占位符名字，我们都必须提供一个单独的using声明。编写这样的声明很烦人。可以这样解决：

```c
//将在18.2.2介绍
//using namespace namespace_name;
//所有来自namespace_name的名字都可以在我们的程序中直接使用
using namespace std::placeholders;
```

#### bind的参数

-   可以用bind绑定给定可调用对象中的参数或重新安排其顺序

```c
//g是一个有两个参数的可调用对象
auto g=bind(f,a,b,_2,c,_1);
/*
映射为f(a,b,_2,c,_1)
例如，调用g(X,Y)会调用f(a,b,Y,c,X)
*/
g(_1,_2);
```

#### 用bind重排参数顺序

```c
//按单词长度由短至长排序
sort(words.begin(),words.end(),isShorter);
//按单词长度由长至短排序
sort(words.begin(),words.end(),bind(isShorter,_2,_1));
```

#### 绑定引用参数

-   为了替换一个引用方式捕获ostream的lambda：  

```c
//os是一个局部变量，引用一个输出流 c是一个局部变量，类型为char
for_each(words.begin,words.end(),[&os,c] (const string &s) { os<<s<<c;});

ostream & print(ostream &os,const string &s,char c)
{
　　os<<s<<c;
}

//错误：不能拷贝os
for_each(words.begin(),words.end(),bind(print,os,_1,' '));
//函数ref返回一个对象，包含给定的引用，此对象是可以拷贝的
for_each(words.begin(),words.end(),bind(print,ref(os),_1,' '));
```

-   除了ref函数，还有cref函数，生成一个保存const引用的类
-   函数ref和cref定义在头文件functional中、

---

## 10.4 再探迭代器

-   参考[《再探迭代器（插入迭代器、流迭代器、反向迭代器、移动迭代器）》](http://www.cnblogs.com/wuchanming/p/3918408.html)
-   除了为每个容器定义的迭代器之外，标准库在头文件iterator中还定义了额外几种迭代器。这些迭代器包括以下几种:
    -   插入迭代器：这些迭代器被绑定到一个容器上，可用来向容器插入元素
    -   流迭代器：这些迭代器被绑定到输入或输出上，可用来遍历所有关联的IO流
    -   反向迭代器：这些迭代器向后而不是向前移动。除了forward_list之外的标准库容器都有反向迭代器
    -   移动迭代器：这些专用的迭代器不是拷贝其中的元素，而是移动它们。将在13.6.2介绍

### 10.4.1 插入迭代器

-   当我们通过一个迭代器进行赋值时，该迭代器调用容器操作来向给定容器的指定位置插入一个元素

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_10_2.png)

-   插入迭代器有三种类型，差异在于元素插入的位置：
    -   `back_inserter`创建一个使用`push_back`的迭代器
    -   `front_inserter`创建一个使用`push_front`的迭代器
    -   inserter创建一个使用insert的迭代器。此函数接受三个参数，这个参数必须是一个指向给定容器的迭代器。元素将被插入到给定迭代器所表示的元素之前。
-   只有在容器支持`push_front`的情况下，我们才可以使用`front_inserter`。类似的，只有在容器支持`push_back`的情况下，我们才能使用back_inserter
-   当调用inserter(c,iter)时，我们得到一个迭代器，接下来使用它时，会将元素插入到iter原来所指的位置之前的位置。

```c
//如果it是由inserter生成的迭代器
*it=val;
//其效果与下面代码一样
it=c.insert(it,val);//it指向新加入的元素
++it; //递增it使它指向原来的元素
```

-   使用front_inserter时，元素总是插入到容器第一个元素之前，即使我们传递给inserter的位置原来指向第一个元素，只要我们在此元素之前插入一个新元素，此元素就不再是容器的首元素了

```c
list<int> lst={1,2,3,4};
list<int> lst2,lst3;  //空list
//拷贝完成之后，lst2包含4 3 2 1,调用的是容器的push_front
copy(lst.begin(),lst.end(),front_inserter(lst2));
//拷贝完成之后lst3包含1 2 3 4 
copy(lst.begin(),lst.end(),inserter(lst3,lst3.begin()));
```

### 10.4.2 iostream迭代器

-   虽然iostream类型不是容器，但标准库定义了用于这些IO类型对象的迭代器
    -   istream_iterator读取输入流
    -   ostream_iterator向一个输出流写数据
-   通过使用流迭代器，我们可以用泛型算法从流对象读取数据以及向其写入数据

#### istream_iterator操作

-   一个`istream_iterator`使用>>来读取流。因此，`istream_iterator`要读取的类型必须定义了输入运算符。
-   可以默认初始化迭代器，这样就创建了一个可以当作尾后值使用的迭代器

```c
istream_iterator<int> int_it(cin); //从cin读取int
istream_iterator<int> int_eof; //尾后迭代器
ifstream in("afile"); 
istream_iterator<string> str_in(in); //从“afile”读取字符串

//下面是一个用istream_iterator从标准输入流读取数据，存入一个vector的例子：
istream_iterator<int> in_iter(cin); //从cin读取int
istream_iterator<int> eof;  //istream尾后迭代器
while(in_iter!=eof)
　　//后置递增运算读取流，返回迭代器的旧值
　　//解引用迭代器，获得从流读取的前一个值
　　vec.push_back(*in_iter++);
```

-   我们可以将程序重写为如下形式，这体现了istream_iterator更有用的地方：

```c
istream_iterator<int> in_iter(cin),eof; //从cin读取int
vector<int> vec(in_iter,eof);  //从迭代器范围构造vec
```

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_10_3.png)

#### 使用算法操作流迭代器

```c
istream_iterator<int> in(cin),eof;
cout<<accumulatre(in,eof,0)<<endl;

//若输入为：1 3 7 9 9 
//输出为：29
```

#### istream_iterator允许使用懒惰求值

-   当我们将一个istream_iterator绑定到一个流时，标准库并不保证迭代器立即从流读取数据。具体实现可以推迟从中读取数据，直到我们使用迭代器时才真正读取。
-   保证的是，在我们第一次解引用迭代器之前，从流中读取数据的操作已经完成了

#### ostream_iterator操作

-   我们可以对任何输出运算符(<<运算符)的类型定义ostream_iterator
-   当创建一个ostream_iterator时，我们可以提供（可选的）第二参数，它是一个字符串，在输出每个元素后都会打印此字符串。此字符串必须是一个C风格字符串（即，一个字符串字面值或者一个指向以空字符结尾的字符数组的指针）

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_10_4.png)

```c
//使用ostream_iterator来输出值的序列
ostream_iterator<int> out_iter(cout," ");
for(auto e:vec)
　　*out_iter++=e;   //赋值语句实际上将元素写到cout
cout<<endl;

//可以忽略解引用和递增运算，以下写法和上面的写法效果一样
//运算符*和++实际上对ostream_iterator对象不做任何事情
for(auto e:vec)
　　out_iter=e;//赋值语句将元素写道cout
cout<<end;

//可以通过调用copy来打印vec中的元素，这比编写循环更为简单：
copy(vec.begin(),vec.end(),out_iter);
cout<<endl;
```

#### 使用流迭代器处理类类型

-   只要类型有输出运算符(<<)，我们就可以为其定义`ostream_iterator`（例如Sales_item）

```c
istream_iterator<Sales_item> item_iter(cin),eof;
ostream_iterator<Sales_item> out_iter(cout,"\n");
Sales_item sum=*item_iter++;
while(item_iter!=eof)
{
    if(item_iter->isbn()==sum.isbn())
        sum+=*item_iter++;
    else
    {
        out_iter=sum;
        sum=*item_iter++;
    }
}
out_iter=sum;
```

### 10.4.3 反向迭代器

-   反向迭代器就是在容器中从尾元素向首元素反向移动的迭代器。对于反向迭代器，递增（以及递减）操作的含义会颠倒过来。递增一个反向迭代器（++it）会移动到前一个元素；递减一迭代器（--it）会移动到下一个元素
-   除了forward_list之外，其他容器都支持反向迭代器
-   调用rbegin、rcend、crbegin和crend成员函数来获得反向迭代器
-   这些成员函数返回指向容器尾元素和首元素之前一个位置的迭代器
-   反向迭代器也有const和非const版本

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/img_10_1.png)

```c
//按“正常序”排序vec
sort(vec.begin(),vec.end());
//按逆序排序：将最小元素放在vec的末尾
sort(vec.rbegin(),vec.rend());
```

#### 反向迭代器需要递减运算符

-   除了forward_list之外，标准容器上的其他迭代器都既支持递增运算又支持递减运算
-   流迭代器不支持递减运算
-   因此，不可能从一个forward_list或一个流迭代器创建反向迭代器

#### 反向迭代器与其他迭代器间的关系

```c
//在一个逗号分隔的列表中查找第一个元素
auto comma=find(line.cbegin(),line.cend(),',');
cout<<string(line.cbegin(),comma)<<endl;

//在一个逗号分隔的列表中查找最后一个元素
auto rcomma=find(line.crbegin(),line.crend(),',');

//错误:将逆序输出单词的字符
//如果我们的输入是:FIRST,MIDOLE,LAST
//则这条语句会打印TSAL
cout<<string(line.crbegin(),rcomma)<<endl;
```

-   我们不能直接使用rcomma。因为它是一个反向迭代器，意味着它会反向朝着string的开始位置移动
-   将rcomma转换回一个普通迭代器，能在line中正向移动
-   通过调用reverse_iterator的base成员函数来完成这一转换，此成员函数会返回其对应的普通迭代器：

```c
cout<<string(rcomma.base(),line.cend())<<endl;
```

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/img_10_2.png)

-   rcomma和rcomma.base()指向了不同的元素
-   这些不同保证了元素范围无论是正向处理还是反向出来都是相同的
-   普通迭代器与反向迭代器的关系反映了左闭合区间的特征
-   关键点在于[line.crbegin(),rcomma)和[rcomma.base(),line.cend())指向line中相同的元素范围

---

## 10.5 泛型算法结构

-   参考[《泛型算法结构》](http://www.cnblogs.com/wuchanming/p/3918411.html)
-   任何算法的最基本的特性是它要求其迭代器提供哪些操作
-   算法所要求的迭代器操作可以分为5个迭代器类别

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_10_5.png)

-   第二种算法分类的方法是按照是否可读、写或是重排序列中的元素来分类，在附录A按照这种分类方法列出了所有算法

### 10.5.1 5类迭代器

#### 迭代器类别

##### 输入迭代器

-   输入迭代器：可以读取序列中的元素。一个输入迭代器必须支持
    -   用于比较两个迭代器的相等和不相等运算符（==、！=）
    -   用于推进迭代器的前置和后置递增运算（++）
    -   用于读取元素的解引用运算符（*）；解引用只会出现在赋值运算符的右侧
    -   箭头运算符（->），等价于（*it）.member，即，解引用迭代器，并提取对象的成员
-   只用于顺序访问
-   不能保证输入迭代器的状态可以保存下来并用来访问元素，因此，输入迭代器只能用于单遍扫描算法
-   算法find和accumulate要求输入迭代器
-   istream_iterator是一种输入迭代器

##### 输出迭代器

-   输出迭代器：可以看做输入迭代器功能上的补集——只写而不读元素。输出迭代器必须支持
    -   用于推进迭代器的前置和后置递增运算（++）   
    -   解引用运算符（*），只能出现在赋值运算符的左侧（向一个已经解引用的输出迭代器赋值，就是将值写入它所指向的元素）
-   只能向一个输出迭代器赋值一次
-   只能用于单遍扫描算法
-   用作目的位置的迭代器通常都是输出迭代器
-   copy函数的第三个参数就是输出迭代器
-   ostream_iterator类型也是输出迭代器

##### 前向迭代器

-   可以读元素
-   这类迭代器只能在序列中沿一个方向移动
-   支持所有输入和输出迭代器的操作
-   可以多次读写同一个元素，因此，我们可以保存前向迭代器的状态
-   可以对序列进行多遍扫描
-   算法replace要求前向迭代器
-   forward_list上的迭代器就是前向迭代器

##### 双向迭代器

-   可以正向/反向读写序列中的元素
-   除了支持所有前向迭代器的操作之外，双向迭代器还支持前置和后置递减运算符（--）
-   算法reverse要求双向迭代器
-   除了forward_list之外，其他标准库都提供符合双向迭代器要求的迭代器

##### 随机迭代器

-   提供在常量时间内访问序列中的任意元素的能力
-   支持双向迭代器的所有功能
-   还支持如下的操作（表3.7中的操作）：
    -   用于比较两个迭代器相对位置的关系运算符（<、<=、>和>=）
    -   迭代器和一个整数值的加减运算（+、+=、-和-=），计算结果是迭代器在序列中前进（或后退）给定整数个元素后的位置
    -   用于两个迭代器上的减法运算符（-）得到两个迭代器的距离
    -   下标运算符（iter[n]，与*（iter[n]）等价
-   算法sort要求随机访问迭代器
-   array、deque、string和vector的迭代器以及用于访问内置数组元素的指针都是随机访问迭代器

### 10.5.2 算法形参模式

-   大多数算法具有如下4种形式之一：

```c
alg(beg,end,other args);
alg(beg,end,dest,other args);
alg(beg,end,beg2,other args);
alg(beg,end,beg2,end2,other args);
```

### 10.2.3 算法命名规范

#### 一些算法使用重载形式传递一个谓词

-   接受谓词参数来代替<或==运算符的算法，以及哪些不接受额外参数的算法，通常都是重载的函数

```C
//重载函数
unique(beg,end);  //使用==运算符比较元素
unique(beg,end,comp); 　　//使用comp比较运元素
```

-   由于两个版本的函数在参数个数上不相等，因此具体应该调用那个不会产生歧义

#### _if版本的算法


-   接受一个元素值的算法通常有另一个不同名的版本，该版本接受一个谓词代替元素值。接受谓词参数的算法都有附加_if前缀

```c
find(beg,end,val); //查找输入范围中val第一次出现的位置
find_if(beg,end,pred);  //查找第一个令pred为真的元素
```

-   这两个算法提供了命名上的差异的版本，而非重载版本，因为两个版本的算法都接受相同数目的参数

#### 区分拷贝元素的版本和不拷贝的版本

-   写到额外目的空间的算法都在名字后面附加一个_copy：

```c
reverse(beg,end); //反转输入范围中元素的顺序
reverse_copy(beg,end,dest); //将元素按逆序拷贝到dest
```

-   一些算法同时提供`_copy`和`_if`版本

```c
//从v1中删除奇数元素
remove_if(v1.begin(),v1.end(),
　　　　　　　　　　　　[](int i) {return i%2;});
//将偶数元素从v1拷贝到v2；v1不变
remove_copy_if(v1.begin(),v1.end(),back_inserter(v2),
　　　　　　　　　　　　[](int i) {return i%2;});
```

---

## 10.6 

-   参考[《特定容器算法》](http://www.cnblogs.com/wuchanming/p/3918418.html)
-   链表类型list与forward_list定义了几个成员函数形式的算法
-   有这些算法的原因：
    -   通用版本的sort要求随机访问迭代器，因此不能用于list和forward_list
    -   链表类型定义的其他算法的通用版本可以用于链表，但代价太高。这些算法需要交换输入序列中的元素。一个链表可以通过改变元素间的链接而不是真正的交换它们的值来传递“交换”元素。因此，这些链表版本的算法的性能比对应的通用版本好很多
-   对于list和forward_list应该优先使用成员函数版本的算法而不是通用算法   

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_10_6_1.png)
![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_10_6_2.png)

#### splice成员

-   链表类型还定义了splice算法。此算法是链表数据结构所特有的

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_10_7.png)

#### 链表特有的操作会改变容器

-   链表特有版本与通用版本间的一个至关重要的区别是:**链表版本会改变底层的容器**。例如:
    -   remove的链表版本会删除指定的元素。unique的链表版本会删除第二个和后继的重复元素
    -   merge和splice会销毁其参数:通用版本的merge将合并的序列写给一个给定的目的迭代器：两个输入序列是不变的；链表版本的merge函数会销毁给定的链表，元素从参数指定的链表中删除，被合并到调用merge的链表对象中。在merge之后，来自两个链表中的元素仍然存在，但它们都已在同一个链表中

---

## 小结

-   算法从不会直接改变它们所操作的序列的大小。它们将元素从一个位置拷贝到另一个位置，但不会直接添加或删除元素
-   容器`forward_list`和list对一些通用算法定义了自己的特有的版本。与通用算法不同，这些链表特有版本会修改给定的链表


