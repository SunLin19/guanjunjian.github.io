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
a2 = {0}; //错误，不能将一个花括号列表赋予数组  
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



