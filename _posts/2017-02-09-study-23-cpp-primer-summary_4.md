---
layout:     post
title:      "「二十三」《C++Primber》笔记 第IV部分"
date:       2017-02-09 09:00:00
author:     "guanjunjian"
categories: 阅读
tags:
    - study
    - summary
---

* content
{:toc}

>
> 《C++Primer》笔记 第IV部分 高级主题
> 
> 包括第17至19章
> 
> [第17章 标准库特殊设施] [第18章 用于大型程序的工具] [第19章 特殊工具与技术]
> 
> 第五版 
>

# 第17章 标准库特殊设施

## 17.1 tuple类型

-   tuple，元组，定义在头文件tuple中
-   当希望将一些数据组合成单一对象，但又不想麻烦地定义一个新数据结构来表示这些数据时，使用tuple
-   可以将tuple看作一个“快速而随意”的数据结构

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_1.png)




### 17.1.1 定义和初始化tuple

-   定义一个tuple时
    -   需要指出每个成员的类型
    -   可以使用tuple的默认构造函数（是explicit的），它会对每个成员进行值初始化
    -   可以为每个成员提供一个初始值
    -   使用make_tuple函数生成tuple对象

```c
//错误，tuple的构造函数是explicit的，必须使用直接初始化语法
tuple<size_t,size_t,size_t> threeD = {1,2,3};
//正确
tuple<size_t,size_t,size_t> threeD{1,2,3};

//使用make_tuple函数
//根据初始值的类型来判断tuple的类型
//类型为tuple<const char*,int,double>
auto item = make_tuple("0-999-78345-X",3,20.00);
```

#### 访问tuple成员 

```c
auto price = get<2>(item); //返回第三个成员
```

-   从0开始计数，即get<0>(item)是第一个成员

---

-   查询tuple成员的数量和类型，其中查类型有2种方法：

```c
//类型查询方法一：tuple整体类型
typedef decltype(item) trans; 
//类型查询方法二：tuple单个成员类型
tuple_element<1,trans>::type cnt = get<1>(item);

//查询数量
size_t sz = tuple_size<trans>::value;
```

#### 关系和相等运算符

-   为了使用关系运算符，对每对成员使用<必须都是合法的

```c
tuple<string,string> duo("1","2");
tuple<size_t,size_t> twoD(1,2);
//错误，不能比较string和size_t
bool b = (duo == twoD);
tuple<size_t,size_t,size_t> threeD(1,2,3);
//错误，成员数量不同也不能比较
b = (twoD < threeD);
tuple<size_t,size_t> origin(0,0);
//正确，b为true
b = (origin < twoD);
```

### 17.1.2 使用tuple返回多个值

```c
typedef tuple<size_t,size_t,size_t> matches;
vector<meches> findbook() {//...}
```

---

## 17.2 bitset类型

-   标准库bitset类，使得位运算的使用更容易，且能处理超过最长整形类型大小的位集合，在头文件bitset中

### 17.2.1 定义和初始化bitset

-   bitset是类模板，具有固定大小
-   定义一个bitset时，需要声明它包含多少个二进制位，位大小必须是一个常量表达式
-   二进制位的位置是从0开始编号的。编号从0开始的二进制位称为**低位**，编号到31位结束的二进制位称为**高位**

```c
bitset<32> bitvce(1U); //32位，低位为1，其他位位0
```

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_2.png)

#### 用unsigned值初始化bitset

-   当我们使用一个**整型值**初始化bitset时，此值将转换为**unsigned long long**类型并被当作位模式来处理，bitset中的二进制位将是此模式的一个副本

```c
//0xbeef为1011111011101111
//bitvec1比初始值小；初始值中的高位被丢弃
bitset<13> bitvec1(0xbeef); //二进制位序列为1111011101111
//bitvec2比初始值大；初始值中的高位被置位0
bitset<20> bitvec2(0xbeef); //二进制位序列为00001011111011101111
//在64位机器中，long long 0ULL是64个0比特，因此~0ULL是64个1
bitset<128> bitvec3(~0ULL); //0~63位为1,63~127位为0
```

#### 从一个string初始化bitset

```
bitset<32> bitvec4("1100"); //二进制位序列为1100
```

-   string的下标编号习惯与bitset相反
    -   string中下标最大的字符（最右字符）用来初始化bitset中的低位（下标为0的二进制位）

-   可以只用一个子串作为初始值

```c
string str("1111111000000011001101");
bitset<32> bitvec5(str,5,4); //从str[5]开始的四个二进制位，1100
//第二个参数为“开始位置”
bitset<32> bitvec6(str,str.size-4); //最后四个字符，1101 
```

### 17.2.2 bitset操作

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_3.png)

-   置位：设为1
-   复位：设为0
-   const版本的下标运算符在指定位置置位时返回true，否则false；非const版本分会bitset定义的一个特殊类型，它允许我们操作指定位的值

```c
bitvec[0] = 0; //将第1位复位
bitvec[0].flip(); //翻转第1位
```

#### 使用bitset

**使用位运算符版本**

```c
bool status;
//此值被当作位集合使用
unsigned long quizA = 0;
quizA |= 1UL << 27; //指出第27个学生通过了测试
status = quizA & (1UL << 27); //检查第27个学生是否通过了测试
quizA &= ~(1UL << 27); //第27个学生未通过测试
```

**使用bitset版本**

```c
bool status;
bitset<30> quizB;  //每个学生都分配一个位，所有位都被初始化为0
quizB.set(27);  //指出第27个学生通过了测试  
status = quizB[27]; //检查第27个学生是否通过了测试
quizB.reset(27); //第27个学生未通过测试
```

---

## 17.3 正则表达式

-   C++正则表达式库（RE库），定义在头文件regex，包含多个组件

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_4.png)

-   如果整个输入序列与表达式匹配，则regex_match函数返回true
-   如果输入序列中一个子串与表达式匹配，则regex_search函数返回true
-   表17.5是`regex_search`和`regex_match`的参数，其中的mtf为匹配标志，将在在表17.13中描述

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_5.png)

### 17.3.1 使用正则表达式库

```c
//查找不在字符串c之后的字符串ei
string pattern("[^c]ei");
//[[:alpha:]]匹配任意字母
pattern = "[[:alpha:]]*" + pattern + "[[:alpha:]]*";
//构造一个用于查找模式的regex
regex r(pattern); 
//定义一个对象，用于保存搜索结果
smatch results;
//定义一个strting保存于模式匹配和不匹配的文本
string test_str = "receipt freind theif receive";
//用r在test_str中查找与pattern匹配的子串
if (regex_search(test_str, results, r))
    cout << results.str() << endl;    
```

-   由于regex_search查找到一个匹配子串就会停止查找，因此程序的输出为：`friend`

-   regex默认使用的正则表达式语言是ECMAScript，其他可选语言可参考表17.6

**ECMAScript正则表达式语言的一些特性：**

-   `\{d}`表示单个数字， `\{d}{n}`表示n个数字的序列。（如，`\{d}{3}`匹配3个数字的序列）
-   在方括号中的字符集表示匹配这些字符中任意一个。（如， [-. ]匹配一个短横线，一个点或一个空格。注意，点在括号中没有特殊含义）
-   后接'?'的组件是可选的。（如，`\{d}{3} [-. ]? \{d}{4}` ,表示开始是3个数字，后接一个可选的短横线或空格，然后是4个数字）
-   ECMAScript使用反斜杠表示字符本身，而不是其特殊含义。由于反斜线也是C++的特殊字符，在模式中每次出现`\`的地方，我们都必须用一个额外的反斜线来告知C++我们需要一个反斜线而不是一个特殊符号。因此我们用`\\{d}{3}`来表示正则表达式`\{d}{3}`
-   [[:alpha:]]匹配任意字母
-   `+`表示“一个或多个”匹配
-   `*`表示“零个或多个”匹配
-   `.`通常匹配任意字符。如果在C++中表达正则表达式的`.`，需要使用`\.`。如果在C++中表达`.`的本身含义，使用需要`\\.`(第1个反斜线去掉C++语言中反斜线的特殊含义，即，正则表达式字符串为`\.`；第2个反斜线表示在正则表达式中去掉.的特殊含义)

---

#### 指定regex对象的选项

-   表17.6中最后6个标志指出编写正则表达式所用的语言，必须设置其中之一，且只能设置一个。默认情况下为ECMAScript

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_6.png)

```c
//一个或多个字幕或数字字符后接一个`.`，再接`cpp`或`cxx`或`cc`
//由于设置了regex::icase，因此会忽略大小写
regex r("[[:alnum:]]+\\.(cpp|cxx|cc)$", regex::icase);
smatch results; 
string filename;
while (cin >> filename) 
	if (regex_search(filename, results, r))
		cout << results.str() << endl; 
```

#### 指定或使用正则表达式时的错误

-   一个正则表达式的语法是否正确是在运行时解剖的
-   在运行时会抛出一个类型为regex_error的异常，可以调用它的以下两个操作
    -   what()：描述发生了什么错误，错误类型在表17.7中
    -   code()：返回某个错误类型应用的数值编码

```c
try {
    //...
}catch(regex_error e)
{
    cout<< e.what() << "\ncode:" << e.code() << endl;
}

//输出为:
regex_error(error_brack):
The expression contained mismatched [ and ].
code: 4
```

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_7.png)

#### 避免创建不必要的正则表达式

-   一个正则表达式所表示的“程序”是在运行时而非编译时编译的。正则表达式的编译是一个非常慢的操作。应该努力避免创建很多不必要的regex

#### 正则表达式类和输入序列类型

-   使用的RE类必须与输入序列类型匹配，表17.8指出了RE库类型与输入序列类型的对应关系

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_8.png)

### 17.3.2 匹配与Regex迭代器类型

-   `regex_search`匹配到第一个子串后就会停止，可以使用`sregex_iterator`来获得所有匹配
-   regex迭代器是一种迭代器适配器，被绑定到一个输入序列和一个regex对象上
-   如表17.8所示，每种不同输入序列类型都有对应的特殊regex迭代器类型
-   迭代器操作如表17.9所述

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_9_1.png)
![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_9_2.png)

-   将一个`sregex_iterator`绑定到一个string和一个regex对象时，迭代器自动定位到给定string中第1个匹配位置；使用递增运算符后，会得到下一个匹配位置（都是调用`regex_search`实现）
-   解引用迭代器，会得到一个smatch对象

#### 使用sregex_iterator

```c
for(sregex_iterator it(file.begin(),file.end(),r), end_it; it!= end_it; ++it)
{
    //it->str()就是调用得到的smatch的str()
    cout << it->str() << endl;
}
```

-   `sregex_iterator end_it`是一个空`sregex_iterator`，起到尾后迭代器的作用

#### 使用匹配数据

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_10.png)

### 17.3.3 使用子表达式

-   正则表达式中的模式（pattern）通常包含一个或多个子表达式。一个子表达式是模式的一部分，本身也具有意义，通常使用括号表示子表达式

```c
//r有两个子表达式：([[:alnum:]]+)和(cpp|cxx|cc)
regex r("([[:alnum:]]+)\\.(cpp|cxx|cc)$", regex::icase);

if(regex_search(filename,results,r))
    cout << results.str(1) << endl;
```

**子表达式的匹配部分**

-   smatch对象还提供访问模式中每个子表达式的能力
-   每一个子表达式的匹配为ssub_match对象（对应string输入）
-   使用[]来访问子匹配，如results[0]表示整个模式对应的匹配的对象，results[1]表示第1个子匹配对象
-   如表17.11所示，str()返回子匹配对象的string。str(0)表示整个模式对应的匹配，随后是每个子表达式对应的匹配。即str(1)表示第1个子表达式对应的匹配，str(2)为第2个子表达式对应的匹配
    -   例如：
    -   foo.cpp
    -   str(0)保存foo.cpp
    -   str(1)保存foo
    -   str(2)保存cpp


---

-   子表达式的常见用途是验证必须匹配特定格式的数据

#### 使用子匹配操作

-   如果一个子表达式是完整匹配的一部分，则其对应的ssub_match对象的matched成员为true

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_11.png)

-   使用如下

```c
bool valid(const smatch& m)
{
	// if there is an open parenthesis before the area code
	if(m[1].matched)
		// the area code must be followed by a close parenthesis
		// and followed immediately by the rest of the number or a space 
	    return m[3].matched 
		       && (m[4].matched == 0 || m[4].str() == " ");
	else 
		// then there can't be a close after the area code 
		// the delimiters between the other two components must match
		return !m[3].matched
		       && m[4].str() == m[6].str();
}
```

### 17.3.4 使用regex_replace

-   在输入序列中查找并替换一个正则表达式时，可以调用`regex_replace`。它接受
    -   输入字符串
    -   regex对象
    -   描述想要的输出形式的字符串

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_12.png)

```c
//将号码格式改为ddd.ddd.dddd
//$后跟子表达式的索引号来表示一个特定的子表达式
string fmt = "$2.$5.$7"; 

regex r(phone);
string number = "(908) 555-1800";
cout << regex_replace(number,r,fmt) << endl;

//输出为：
908.555.1800
```

#### 用来控制匹配和格式的标志

-   匹配和格式化标志的类型为`match_flag_type`，定义在`regex_constants`的命名空间中，而`regex_constants`命名空间又定义在std命名空间中，所以要使用`match_flag_type`，用如下两种方法：

```c
using std::regex_constants::format_no_copy;

using namespcae std::regex_constants;

regex_replace(s,r,fmt2,format_no_copy);
```

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_13.png)

---

## 17.4 随机数

-   C和C++都依赖于一个简单的C库函数rand来生成随机数，但存在一些问题。如需要随机浮点数、非均匀分布的数，为了获得这些数需要转换rand生成的随机数的范围，这常常会引入非随机性
-   定义在头文件random中的随机数库通过一组协助的类来解决上面的问题：
    -   随机数引擎类
    -   随机数分布类

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_14.png)

-   C++程序不应该使用库函数rand，而应使用`default_random_engine`类和恰当的分布类对象

### 17.4.1 随机数引擎和分布

#### 随机数引擎

-   随机数引擎是函数对象类，定义了一个调用运算符，不接受参数并返回一个随机unsigned整数

```c
//生成随机无符号数
default_random_engine e; 
//e()“调用”对象来生成下一个随机数
cout << e() << " ";
```

-   标准库定义了多个随机数引擎，区别在于性能和随机性质量不同。引擎类型列在书附录A.3.2（P783)中

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_15.png)

#### 分布类型和引擎

-   为了得到一个指定范围内的数，我们使用一个分布类型的对象：

```c
//生成0到9之间（包含）均匀分布的随机数
uniform_int_distribution<unsigned> u(0,9);
//生成无符号随机整数
default_random_engine e;
//注意，传入的是e本身，而不是e()
cout << u(e) << endl;
```

-   分布类型也是函数对象类，定义了一个调用运算符，它接受一个随机数引擎作为参数。分布对象使用它的引擎参数生成随机数，并将其映射到指定的分布
-   注意，传递给分布对象的是引擎对象本身，即u(e)。因为某些分布可能需要调用引擎多次才能得到一个值，所以不应该传入引擎生成的一个值，即u(e())
-   **随机数发生器**是指**分布对象**和**引擎对象**的组合

---

-   一个给定的随机数发生器一直会生成相同的随机数序列。一个函数如果定义了局部的随机数发生器，应该将其（包括引擎和分布对象）定义为static的。否则，每次调用函数都会生成相同的序列

#### 设置随机数发生器发生器种子

-   随机数发生器会生成相同的随机序列这一特性在调试中很有用
-   如果希望每次运行程序都会生成不同的随机结果，可以通过提供一个种子来达到这个目的
-   种子是一个数组，可以利用它从序列中一个新位置重新开始生成随机数
-   设置种子的两种方式：
    -   创建引擎时提供种子
    -   调用引擎的seed成员

```c
//使用默认种子
default_random_engine e1;
//使用给定的种子值
default_random_engine e2(2147483646);
//使用默认种子值
default_random_engine e3;
//使用seed设置一个新种子值
e3.seed(32767);
//e3和e4有相同的种子，它们将生成相同的序列
default_random_engine e4(32767);
```

-   选择一个好的种子：调用系统函数time。
-   time
    -   定义在头文件ctime中
    -   返回从某个特定时刻到当前经过了多少秒。由于以秒计时，所以只适用于生成种子的间隔为秒级或更长的应用
    -   接受单个指针参数，指向用于写入时间的结构。如果此指针为空，则函数简单地返回时间。

### 17.4.2 其他随机数分布

-   需要不同类型或不同分布的随机数，通过定义不同**随机数分布对象**来实现

#### 生成随机实数

-   最常用但不正确的方法：rand()的结果除以RAND_MAX。不正确因为：随机整数的精度通常低于随机浮点数，这样，有一些浮点数就永远不会被生成了
-   正确的方法：`uniform_real_distribution`类型的对象

```c
default_random_engine e;
//0到1（包含）的均匀分布
uniform_real_distribution<double> u(0,1);
cout << u(e) << endl;
```

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_16.png)

#### 使用分布的默认结果类型

-   分布类型都是模板，表示分布生成的随机数的类型
-   希望使用默认随机数类型时需要在模板名之后使用空尖括号

```c
//默认生成double
uniform_real_distribution<> u(0,1);
```

#### 生成非均匀分布的随机数

-   可以生成分均匀分布的随机数

```c
default_random_engine e;
normal_distribution<> n(4,1.5);  //均值4，标准差1.5
```

#### bernoulli_distribution类

-   `bernoulli_distribution`类不接受模板参数，是一个普通类，而非模板
-   返回一个bool值
-   返回true的概率是一个常数，此概率的默认值是0.5

```c
//默认是50/50
bernoulli_distribution b;

//设置为 55/45的概率
bernoulli_distribution b(.55);
```

## 17.5 IO库再探

### 17.5.1 格式化输入和输出

-   标准库定义了一组操作符来修改流的格式状态
-   操作符也返回它所处理的刘对象，因此我们可以在一条语句中组合操作符和数组
-   操作符分为两大类输出控制：**控制数值的输出形式**和**控制补白的数量和位置**
-   当操作符改变流的格式状态时，通常改变后的状态对所有后续IO都生效。即，会影响下一个和随后所有的输出，直至另一个操作符又改变了格式为止
    -   例外：setw、endl。不改变输出流的内部状态，它只决定下一个输出
-   set*开头的操作符，接受参数，定义在iomanip中。其他操作符定义在iostream中
-   以下为常用的操作符：

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_17_1.png)
![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_17_2.png)

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_18.png)

### 操作符使用方法

#### 控制布尔值的格式

```c
//以boolalpha为例
bool bool_val = get_status();
cout << boolalpha << bool_val << noboolalpha;
```

#### 指定整型值的进制

-   操作符hex、otc、dec只影响整型运算对象，浮点值的表示形式不受影响（浮点值的由fixed等几个操作符控制）   

```c
cout << hex << 20 << endl;
```

#### 精确打印精度

-   precision成员是重载的
    -   一个版本接受一个int，将精度设为此值，并返回旧精度值
    -   一个版本不接受参数，返回当前精度值
-   setprecision接受一个参数，用来设置精度   

```c
//setprecision、precision
cout << cout.precision(11) << sqrt(2.0) << endl;
cout << setprecision(12) << sqrt(2.0) << endl;
cout << cout.precision() << sqrt(2.0) << endl;
```

### 17.5.2 未格式化的输入/输出操作

-   标准库提供了一组低层操作，支持未格式化IO

#### 单字节操作

-   每次一个字节地处理流
-   它们会读取而不是忽略空白符

```c
char ch;
while(cin.get(ch))
    cout.put(ch);
```

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_19.png)

---

**将字符放回输入流**

-   标准库提供了三种方法退回字符，它们有着细微的差别：
    -   peek返回输入流中下一个字符的副本，但不会将它从流中删除，peek返回的值仍然留在流中
    -   unget使得输入流向后移动，从而最后读取的值又回到流中。即使我们不知道最后流中读取什么值，仍然可以调用unget
    -   putback是更特殊版本的unget：它退回从流中读取的最后一个值，但它接受一个参数，此参数必须与最后读取的值相同
-   在读取下一个值之前，标准库保证我们可以退回最多一个值。即，标准库不保证在中间不进行读取操作的情况下能连续调用putchar或unget

---

**从输入操作读取返回的int**

-   函数peek和无参的get都以int类型从输入流返回一个字符。原因是：可以返回文件尾标记。char范围中的每个值都用来表示一个真实字符，因此取址范围内没有额外的值可以用来表示文件尾，标准库使用负值表示文件尾

```c
int ch;
while((ch=cin.get()) != EOF)
    cout.put(ch);
```

-   一个编程常见的错误是将get或peek的返回值赋予一个char而不是int
-   在一台char被实现为unsigned char的机器上，下面的循环永远不会停止
    -   因为get返回EOF时，此值被转换为一个unsigned char（非负）。转换得到的值与EOF（负）的int不再相等，因此循环永远不会停止   

```c
char ch;
while((ch=cin.get()) != EOF)
    cout.put(ch);
```

-   在一台char被实现为signed char的机器上，我们不能确定循环的行为
-   低层IO通常用于读取二进制值的场合，而这些二进制值不能直接映射到普通字符和数值

---

#### 多字节操作

-   这些操作要求我们自己分配并管理用来保存和提取数据的字符数组

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_20_1.png)
![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_20_2.png)

---

**get与getline**

-   相同：两个函数都一直读取数据，直到下面条件之一发生
    -   已经读取了size-1个字符
    -   遇到了文件尾
    -   遇到了分隔符（delim）
-   不同：
    -   get将分隔符留作istream中的下一个字符
    -   getline读取并丢弃分隔符（delim）
    -   无论哪个函数都不会将分隔符保存在sink中

---

**读取了多少个字符**

-   如果在调用gcount之前调用了peek、unget、putback，则gcount返回值为0

---

### 17.5.3 流随机访问

-   各种流类型通常都支持流中数据的随机访问
-   在大多数系统中，绑定到cin、cout、cerr、clog的流不支持随机访问。可以调用seek和tell函数，但在运行时会出错，将流置于一个无效状态
-   由于istream和ostream类型通常不支持随机访问，所以本节剩余内容只适用于fstream和sstream类型

#### seek和tell函数

-   分为两对函数：
    -   用于输入流，后缀为g，即get的缩写
    -   用于输出流，后缀为p，即put的缩写

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_17_21.png)

-   表中pos类型为`pos_type`，off的类型为`off_type`，这两个类型都是机器相关的。它们定义在头文件istream和ostream中
-   tellg和tellp返回一个`pos_type`

#### 只有一个标记

-   一个流中只维护单一的标记：并不存在独立的读标记和写标记
-   由于只有单一的标记，因此我们只要在读写操作间切换，就必须进行seek操作来重定位标记

---

# 第18章 用于大型程序的工具

## 18.1 异常处理

-   异常处理机制允许程序中独立开发的部分能够在运行时就出现的问题进行通信并做出相应的处理，使得我们能够将问题的检测与解决过程分离开来

### 18.1.1 抛出异常

-   通过抛出一条表达式来引发
-   被抛出的**表达式的类型**以及当前的**调用链**共同决定了哪段处理代码将被用来处理该异常
-   当执行一个throw时，跟在throw后面的语句将不再被执行，程序的控制权从throw转移到与之匹配的catch模块
-   匹配的catch模块可能是：
    -   同一个函数中的局部catch
    -   位于直接或间接调用了发生异常的函数的另一个函数中
-   throw通常作为条件语句的一部分或者作为某个函数的最后（或者唯一）一条语句

#### 栈展开

-   当抛出一个异常后，程序暂停当前函数的执行过程并立即开始寻找与异常匹配的catch子句
-   **栈展开**过程：当throw出现在一个try语句块内时，检查与该try块关联的catch子句
    -   如果找到了匹配的catch，就使用该catch处理异常
    -   如果这一步没找到匹配的catch且该try语句嵌套在其他try块中，则继续检查与外层try匹配的catch子句
    -   如果还是找不到匹配的catch，则退出当前的函数，在调用当前函数的外层函数中继续寻找
    -   最后，找不到匹配的catch，程序将调用标准库函数terminate，终止程序的执行过程
-   栈展开过程沿着嵌套函数的调用链不断查找，直到找到了与异常匹配的catch子句为止；或者也可能一直没找到匹配的catch，则退出主函数后查找过程终止
-   假设找到了一个匹配的catch语句，则程序进入该子句并执行其中的代码。当执行完这个catch子句后，找到该try块关联的最后一个catch子句之后的点，并**从这里继续执行**

#### 展开过程中对象将自动销毁

-   如果在栈展开过程中退出了某个块，编译器将负责确保在这个块中创建的对象能被正确地销毁

#### 析构函数与异常

-   析构函数总是会被执行的，但是函数中负责释放资源的代码却可能被跳过
-   如果析构函数需要执行某个可能抛出异常的操作，则该操作应该被放置在一个try语句块当中，并且在构造函数内部得到处理

#### 异常对象

-   编译器使用异常抛出表达式来对异常对象进行**拷贝初始化**
-   throw语句中的表达式必须拥有**完全类型**
-   如果该表达式是**类类型**的话，则相应的类必须含有一个可访问的**构造函数**和一个可访问的**拷贝或移动构造函数**
-   如果该表达式是**数组**类型或**函数**类型，则表达式将被转换成与之对应的**指针**类型
-   如果退出了某个块，则同时释放块中局部对象使用的内存。因此，**抛出**一个指向**局部对象的指针**几乎是一种**错误**的行为
-   抛出一条表达式时，该表达式的**静态编译类型**决定了一次对象的类型。即，在继承体系中，只有基类部分被抛出（因为是拷贝初始化）

### 18.1.2 捕获异常

-   catch子句捕获异常
-   如果catch无须访问抛出的表达式的话，则我们可以忽略捕获形参的名字
-   声明的类型决定了处理代码所能捕获的异常类型。这个类型必须是完全类型，它可以是左值引用，但不能是右值引用
-   如果catch的参数类型是：
    -   非引用：该参数是异常对象的一个副本
    -   引用，该参数是异常对象的一个别名
-   如果catch的参数是基类类型，可以使用其派生类类型的异常对象进行初始化，如果是：
    -   非引用类型：异常对象呗切掉一部分
    -   是基类的引用：参数以常规方式绑定到异常对象上
-   异常声明的静态类型将决定catch语句所能执行的操作

#### 查找匹配的处理代码

-   最终找到的catch未必是异常的最佳匹配，而是第一个与异常匹配的catch。因此越是专门的catch越应置于整个catch列表的前端
-   对于继承体系，派生类异常的处理代码应该出现在基类异常的处理代码之前

---

-   绝大多数类型转换都不被允许，要求异常的类型和catch声明的类型是精确匹配的
-   允许的类型转换：
    -   从非常量向常量转换，即，一条非常量对象的throw语句可以匹配接受常量引用的catch语句
    -   派生类向基类的类型转换
    -   数组被转换成数组（元素）类型的指针
    -   函数被转换成指向该函数类型的指针
-   不允许的类型转换：
    -   算术类型转换
    -   类类型转换
    -   以及其他所有转换规则

#### 重新抛出

-   一条catch语句通过重新抛出（rethrowing）的操作将异常传递给另外一个catch语句
-   重新抛出仍然是一条throw语句，只不过不包含任何表达式

```c
throw;
```

-   空的throw语句只能出现在catch语句或catch语句直接或间接调用的函数之内
-   如果在处理代码之外的区域遇到了空throw语句，编译器将调用terminate
-   一个重新抛出语句并不指定新的表达式，而是将当前的异常对象沿着调用链向上传递（抛到外层catch）
-   只有当catch异常声明是引用类型时我们对参数做的改变才会被保留并继续传播

#### 捕获所有异常的处理代码

-   为了一次性捕获所有异常，我们使用省略号作为异常声明，这样的处理代码称为捕获所有异常的处理代码
-   一条捕获所有异常的语句可以与任意类型的异常匹配
-   catch(…)可以单独出现，也能与其他catch语句一起出现
-   如果catch(…)与其他几个catch语句一起出现，则catch(…)必须在最后位置。出现在捕获所有异常语句后面的catch语句将永远不会被匹配

```c
void manip() {
    try{
        
    }
    catch(...)
    {
        
    }
}
```

### 18.1.3 函数try语句块与构造函数

-   初始值抛出异常时构造函数体内的try语句块还未生效，所以构造函数体内的catch语句无法处理构造函数**初始值列表**的异常，应该使用**函数try语句块**
-   与**函数try语句块**关联的catch既能处理
    -   **函数体**抛出的异常
    -   成员**初始化列表**抛出的异常

```c
//函数try语句块
template <typename T>
Blob<T>::Blob(std::initializer_list<T> il) try: data(std::make_shared<std::vector<T>(il)) 
{
    //...
}
catch (const std::bad_alloc &e)
{
    handle_out_of_memory(e);
}
```

-   然而，初始化构造函数的参数发生异常，不属于函数try语句块的一部分，该异常属于调用表达式的一部分，并将在调用者所在的上下文中处理

---

**小总结**

-   构造函数参数异常：函数调用者上下文中处理
-   构造函数初始化列表：函数try语句块
-   构造函数体内的try语句块异常：与该体内try关联的catch处理

### 18.1.4 noexcept异常说明

-   noexcept**说明符**和noexcept**运算符**
    -   跟在函数参数列表后是异常说明符
    -   作为noexcept异常说明的bool实参出现时，是一个运算符
-   noexcept说明指定某个函数不会抛出异常

```c
void recoup(int) noexcept;  // 不会抛出异常
void alloc(int);        // 可能抛出异常
```

-   noexcept说明要么出现在该函数的所有声明语句和定义语句中，要么一次也不出现
-   noexcept说明符的位置
    -   函数的尾置返回类型之前
    -   在const及引用限定符之后（成员函数中）
    -   在final、override或虚函数的=0之前（成员函数中）
-   可以在函数指针的声明和定义中指定noexcept
-   在typedef或类型别名中则不能出现noexcept

#### 违反异常说明

-   通常情况下，编译器不会也不必在编译时验证异常说明，即在说明noexcept的同时又含有throw语句，不会报错
-   一旦一个noexcept函数抛出了异常，程序就会调用terminate以确保遵守不在运行时抛出异常的承诺
-   noexcept可以用在两种情况下：
    -   确认函数不会抛出异常
    -   根本不知道该如何处理异常，在这种情况下抛出异常，程序会调用terminate结束程序

#### 向后兼容：异常说明

```c
//以下两个声明等价
void recoup(int) noexcept；
void recoup(int) throw();
```

#### 异常说明的实参

-   noexcept说明符接受一个可选的实参，该实参必须能转换为bool类型
    -   如果实参是true，则函数不会抛出异常
    -   如果实参是false，则函数可能抛出异常

```c
void recoup(int) noexcept(true);    // recoup不会抛出异常
void alloc(int) noexcept(false);    // alloc可能抛出异常
```

#### noexcept运算符

-   noexcept运算符是一个一元运算符，它的返回值是一个bool类型的右值常量表达式，用于表示给定的表达式是否会抛出异常
-   noexcept运算符不会求其运算对象的值

```c
//如果recoup不抛出异常则结果为true；否则结果为false
noexcept(recoup(i));
//当e调用的所有函数都做了不抛出说明且e本身不含有throw语句时，返回true，否则false
noexcept(e)
```

-   也使用noexcept运算符得到如下的异常说明：

```c
//f和g的异常说明一致，即都是noexcept(true)或noexcept(false)
void f()  noexcept( noexcept(g()));
```

#### 异常说明与指针、虚函数和拷贝控制

-   **函数指针**及该指针所指的函数必须具有一致的异常说明
    -   指针做了不抛出异常的声明，则该指针只能指向不抛出异常的函数
    -   如果我们显式或隐式地说明了指针可能抛出异常，则该指针可以指向任何函数，即使承诺了不抛出异常的函数也可以

-   **虚函数**  
    -   如果一个虚函数承诺了它不会抛出异常，则后续派生出来的虚函数也必须做出同样的承诺
    -   如果基类的虚函数允许抛出异常，则派生类的对应函数既可以允许抛出异常，也可以不抛出异常

-   当编译器合成拷贝控制成员时
    -   如果合成成员调用的任意一个函数可能抛出异常时，则合成的成员是noexcept(false)
    -   如果我们定义了一个析构函数但是没有为它提供异常说明，则编译器将合成一个异常说明。合成的异常说明将与假设由编译器为类合成析构函数时所得的异常说明说明一致

### 18.1.5 异常类层次

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/img_18_1.png)

-   exception拥有what虚函数，负责返回初始化异常对象的信息

---

-   面向应用的异常类继承自标准异常类

```c
class isbn_mismatch::public std::logic_error{
    //...
}
```

-   使用自定义异常类的方式与使用标准异常类的方式完全一样

```c
throw isbn_mismatch("wrong isbns",isbn(),rhs.isbn());
```

---

## 18.2 命名空间

-   多个库将名字放置在全局命名空间中将引发命**名空间污染**
-   命名空间为防止名字冲突提供了更加可控的机制
-   命名空间分割了全局命名空间，其中每个命名空间是一个作用域

### 18.2.1 命名空间定义

```c
namespace cplusplus_primer {
    //...
} //命名空间结束后无需分号
```

-   命名空间的名字也必须在定义它的作用域内保持唯一
-   命名空间可以定义在：
    -   全局作用域内
    -   其他命名空间中
-   命名空间不能定义在：
    -   函数内部
    -   类内部

#### 每个命名空间都是一个作用域

-   命名空间中的每个名字都必须表示该空间内的唯一实体
-   在不同命名空间中可以有相同的成员
-   定义在某个命名空间中的名字可以被该命名空间内的其他成员直接访问，也可以被这些成员内嵌作用域中的任何单位访问
-   位于该命名空间之外的代码则必须指出所用的名字属于哪个空间

#### 命名空间可以是不连续的

-   命名空间可以定义在几个不同的部分

```c
namespace nsp {
    //...
}
```

-   这里nsp可以表示：
    -   定义了一个名为nsp的新命名空间
    -   为已经存在的命名空间nsp添加新成员

-   不连续的特性可以将接口和实现分离
-   不把`#include`函数库放在命名空间内部，如`#include<string>`放在namespace内部意味着试图将命名空间std嵌套在自定义的命名空间中

#### 定义命名空间成员

-   方法一：命名空间中定义成员

```c
#include "Sales_data.h"
//重新打开命名空间cplusplus_primer
namespace cplusplus_primer {
    //命名空间中定义的成员可以直接使用名字，此时无须前缀
    std::istream& operator>>(...){}
}
```

-   方法二：在命名空间定义的外部定义该命名空间的成员
    -   尽管命名空间的成员可以定义在命名空间外部，但是这样的定义必须出现在所属命名空间的外层空间中。不能在一个不相关的作用域中定义这个运算符  

```c
//命名空间之外定义的成员使用含有前缀的名字
cplusplus_primer::Sales_data
cplusplus_primer::operator+(...){}
```

#### 模板特例化

-   模板特例化必须定义在原始模板所属的命名空间中
-   只要我们在命名空间中声明了特例化，就能在命名空间外部定义它了

```c
//我们必须将模板特例化声明成std的成员
namespace std {
    template<> struct hash<Sales_data>;
}

//在std中添加了模板特例化的声明后，就可以在命名空间std的外部定义它了
template<> struct std::hash<Sales_data>
{
    //...
}
```

#### 全局命名空间

-   全局作用域中定义的名字（即在所有类、函数及命名空间之外定义的名字）也就是定义在全局命名空间中
-   全局命名空间以隐式的方式声明，并且在所有程序中都存在
-   全局作用域中定义的名字被隐式地添加到全局命名空间中
-   全局命名空间没有名字，用以下方法表示使用全局命名空间的成员

```c
::member_name
```

#### 嵌套的命名空间

-   嵌套的命名空间是指定义在其他命名空间中的命名空间
-   内层命名空间的名字将隐藏外层命名空间中的同名成员

#### 内联命名空间

-   C++11新标准引入了一种新的嵌套命名空间，称为内联命名空间
-   无须在内联命名空间的名字前添加表示该命名空间的前缀，通过外层命名空间的名字就可以直接访问它
-   关键字inline必须出现在命名空间第一次定义的地方，后续再打开命名空间的时候可以写inline也可以不写

```c
//创建FifthEd内联命名空间
inline namespace FifthEd {
    //...
}

//再次打开，隐式内联
namespace FifthEd {
    //...
}
```

#### 未命名的命名空间

-   未命名的命名空间是指关键字namespace后紧跟花括号起来的一系列声明语句
-   未命名的命名空间中定义的变量拥有**静态生命周期**：它们在第一次使用前创建，并且直到程序结束才销毁
-   一个未命名的命名空间可以在某个给定的文件内不连续，但是不能跨越多个文件。每个文件定义自己的未命名的命名空间
-   如果一个头文件定义了未命名的空间，则该命名空间中定义的名字将在每个包含了该头文件的文件中对应**不同实体**
-   定义在未命名的命名空间中的名字可以直接使用，不能对未命名的命名空间的成员使用作用域运算符
-   未命名的命名空间中定义的名字的作用域与该命名空间所在的作用域相同
-   如果未命名的命名空间定义在文件的最外层作用域中，则该命名空间中的名字一定要与全局作用域中的名字有所区别

```c
int i;  // i的全局声明
namespace 
{
    int i;
}
// 二义性：i的定义既出现在全局作用域中，又出现在未嵌套的未命名的命名空间中
i = 10;
```

-   一个未命名的命名空间也能嵌套在其他命名空间中，未命名的命名空间中的成员可以通过外层命名空间的名字来访问

```c
namespace local {
    namespace {
        int i;
    }
}
//正确，定义在嵌套的未命名空间中的i与全局作用域中的i不同
local::i = 42;
```

**未命名的命名空间取代文件中的静态声明**

-   旧时，需要将名字声明成static的以使得其对整个文件有效
-   现在，使用未命名的命名空间

### 18.2.2 使用命名空间成员

-   有以下方法：
    -   命名空间的别名
    -   using声明
    -   using指示

#### 命名空间的别名

-   命名空间的别名使得我们可以为命名空间的名字设定一个短得多的同义词

```c
namespace cplusplus_primer { }

//命名空间的别名
namespace primer = cplusplus_primer

//也可以指向一个嵌套的命名空间
namespace Qlib = cplusplus_primer::Querrylib;
Qlib::Query q;
```

#### using声明

-   一条using声明(usingdeclaration)语句一次只引入命名空间的一个成员
-   一条using声明**可以**出现在：
    -   全局作用域
    -   局部作用域
    -   命名空间作用域
    -   类的作用域：在类的作用域中，这样的声明语句只能指向**基类成员**

#### using指示

-   using指示可以使用命名空间名字的简写形式
-   using指示使得某个特定的命名空间中的所有名字都可见
-   一条using指示**可以**出现在：
    -   全局作用域
    -   局部作用域
    -   命名空间作用域
-   一条using指示**不可以**出现在：
    -   类的作用域中  

**using声明和using指示的作用域**

-   **using声明**的名字的作用域与using声明语句本身的作用域一致
-   对于using声明来说，我们只是简单地令名字在局部作用域内有效

---

-   **using指示**具有将命名空间成员提升到包含命名空间本身和using指示的最近作用域的能力
-   using指示是令整个命名空间的所有内容变得有效
-   using指示一般被看作是出现在最近的**外层作用域**中
-   下例中，A中的名字仿佛出现在全局作用域中f之前的位置一样，即，A中的名字注入到全局作用域中

```c
//命名空间A和函数f定义在全局作用域中
namespace A {
    int i,j;
}

void f()
{
    using namespace A; //把A中的名字注入到全局作用域中
    cout << i*j << endl; //使用命名空间A中的i和j
}
```

---

**using指示示例**

```c
namespace blip {
    int i = 16, j = 15, k = 23;
	//...
}

int j = 0;  // 正确，blip的j隐藏在命名空间中

void manip() 
{
    // using只是，blip中的名字被“添加”到全局作用域中 
	// 如果使用了j，则将在::j和blip::j之间产生冲突
    using namespace blip; 

    ++i;        // 将blip::i设为17
    ++j;      // 二义性错误，是全局的j还是blip::j
    ++::j;    //正确，将全局的j加1
    ++blip::j;  // 正确，将blip::j加1
    int k = 97; // 正确，当前局部的k隐藏了blip:k
    ++k;        // 正确，将当前局部的k加1
}
```

-   blip的成员看起来好像是定义在blip和manip所在的作用域一样
-   并列关系：
    -   当命名空间被注入到它的外层作用域后，很有可能该命名空间中定义的名字会与其外层作用域中的成员冲突。这种冲突是允许的，可以使用::来消除二义性
-   内外层关系：
    -   因为manip的作用域和命名空间的作用域不同，所以manip内部的声明可以隐藏命名空间中的某些成员名字

---

#### 头文件与using声明或指示

-   头文件如果在其顶层作用域中含有using指示或using声明，则会将名字注入到所有包含了该头文件的文件中
-   通常情况下，头文件应该只负责定义接口部分的名字，而不定义实现部分的名字。因此，头文件最多只能在它的函数或命名空间内使用using指示或using声明

### 18.2.3 类、命名空间与作用域

```c
namespace A {
    int i;
    int k;
    class C1 {
    public:
        C1():i(0),j(0) { } //正确，初始化C1::i和C1::j
        int f1() {return k;} //返回A::k
        int f2() {return h;} //错误,h未定义
        int f3();
    private:
        int i; //在C1中隐藏了A::i
        int j;
    };
    int h = i; //正确，用A::i进行初始化
}

//成员f3定义在C1和命名空间A的外部
//正确，返回A::h
int A::C1::f3() {return h;}
```

-   除了类内部出现的成员函数定义外，总是向上查找作用域
-   因为f3的定义位于A::h之后，所以f3对于h的使用是合法的
-   f3按着A::C1::f3的逆序查找h

#### 实参相关的查找与类类型形参

```c
std::string s;
std::cin >> s;

//第二句相当于
operator>>(std::cin,s);
```

-   operator>>函数定义在string中，而string定义在命名空间std中，但不需要为operator>>使用std::和using声明就可以直接调用
-   当我们给函数传递一个类类型的对象时，除了在常规的作用域查找外还会查找实参类所属的命名空间（对于传递类的引用或指针的调用同样有效）
-   因此，在这个例子中，编译器还会查找cin和s的类所属的命名空间，即std，最终在std中找到了operator>>
-   查找规则的这个例外运行概念上作为类接口一部分的非成员函数无须单独的using声明就能被程序使用

#### 查找与std::move和std::forward

-   如果在应用程序中定义了一个标准库中已有的名字，将出现以下两种情况：
    -   根据一般的重载规则确定某次调用应该执行函数的哪个版本
    -   应用程序不会执行函数的标准库版本

#### 友元声明与实参相关的查找

-   当类定义了一个友元时，该友元声明并没有是的友元本身可见
-   一个另外的未声明的类或函数如果第一次出现在友元声明中，则我们认为它（类或函数）是最接近的外层命名空间的成员
-   结合以上两条规则，有

```c
namespace A{
    class C{
        //两个友元，在友元声明之外没有其他的声明
        //这些函数隐式地称为命名空间A的成员
        friend void f2(); //除非另有声明，否则不会被找到
        friend void f(const C&); //根据实参相关的查找规则可以被找到
    };
}

int main()
{
    A::C cobj;
    f(cobj); //正确，通过在A::C中的友元声明查找到A::f
    f2(); //错误：A::f2没有被声明
}
```

-   f接受一个类类型的实参，而f在C所属的命名空间进行了隐式的声明，所以f能被找到

### 18.2.4 重载与命名空间

#### 与参数相关的查找与重载

-   我们将在每个实参类（以及实参类的基类）所属的命名空间中搜索候选函数。在这些命名空间中所有与被调用函数同名的函数都将添加到候选集中，即使其中某些函数调在调用语句处不可见也是如此

```c
namespace NS{
    class Quote{};
    void display(const Quote&){}
}

class Bulk_item:public NS::Quote {};

int main()
{
    Bulk_item book1;
    //调用NS::display
    display(book1);
    return 0;
}
```

#### 重载与using声明

-   using声明语句声明的是一个名字，而非一个特定的函数
-   写using声明时，该函数的所有版本都被引入到当前作用域中

```c
using NS::print(int); //错误，不能指定形参列表
using NS::print; //正确，using声明只声明一个名字
```

-   一个**using声明**引入的函数将重载该声明语句所属作用域中已有的其它同名函数
    -   如果using声明出现在局部作用域中，则引入的名字将隐藏外层作用域的相关声明
    -   如果using声明所在的作用域中已经有一个函数或新引入的函数**同名且形参列表相同**，则该using声明将引发**错误**
    -   using声明将为引入的名字添加额外的重载实例，并最终扩充候选函数集的规模

#### 重载与using指示

-   using指示将命名空间的成员提升到外层作用域中，如果命名空间的某个函数与该命名空间所属作用域的函数同名，则命名空间的函数将被添加到重载集合中
    -   using指示引入一个与已有函数**形参列表完全相同**的函数将**不会产生错误**。此时，只要我们指明调用的是命名空间中的函数版本还是当前作用域的版本即可

```c
namespace libs_R_us {
    extern void print(int);
    extern void print(double);
}
//普通的声明
extern void print(const std::string &);
//这个using指示把名字添加到print调用的候选函数集
using namespace libs_R_us;
//print调用此时的候选函数包括：
//libs_R_us::print(int)
//libs_R_us::print(double)
//print(const std::string &)

void fooBar(int val)
{
    //调用print(const std::string &)
    print("Value:");
    //调用libs_R_us::print(int)
    print(val);
}
```

#### 跨越多个using指示的重载

-   如果存在多个using指示，则来自每个命名空间的名字都会成为候选函数集的一部分

---

## 18.3 多重继承与虚继承

-   多重继承是指从多个直接基类中产生派生类的能力，继承了所有父类的属性

### 18.3.1 多重继承

```c
class Bear: public ZooAnimal {};
class Panda: public Bear, public Endangered {}
```

-   对于派生类能够继承的基类个数，C++没有进行特殊规定；但是在某个给定的派生列表中，同一个基类只能出现一次

#### 多重继承的派生类从每个基类中继承状态

-   派生类的对象包含有每个基类的子对象

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/img_18_2.png)

#### 派生类构造函数初始化所有基类

-   构造一个派生类的对象将同时构造并初始化它的所有基类子对象
-   多重继承的派生类的构造函数初始值也只能初始化它的直接基类
-   派生类的构造函数初始值列表将实参分别传递给每个直接基类
-   基类的构造顺序与**派生列表**中基类的出现顺序保持一致，与派生类**构造函数初始值列表**中基类的顺序无关

#### 继承的构造函数与多重继承

-   允许派生类从它的一个或几个基类中继承构造函数。但如果从多个基类中继承了相同的构造函数（即形参列表完全相同），则程序将产生错误

```c
struct Base1{
    Base1() = default;
    Base1(const std::string&);
    Base1(std::shared_ptr<int>);
};
struct Base2{
    Base2() = default;
    Base2(const std::string&);
    Base2(int);
};

//错误，D1试图从两个基类中都继承D1::D1(const std::string&)
struct D1 : public Base1, public Base2{
    using Base1:Base1; //从Base1继承构造函数
    using Base2:Base2; //从Base2继承构造函数
};
```

-   如果一个类从它的多个基类中继承了相同的构造函数，则这个类必须为该构造函数定义它自己的版本‘

```c
struct D2 : public Base1, public Base2{
    using Base1:Base1; //从Base1继承构造函数
    using Base2:Base2; //从Base2继承构造函数
    //D2必须自定义一个接受string的构造函数
    D2(const string& s):Base1(s),Base(2) {}
    //一旦D2定义了它的构造函数，则必须出现
    D2() = default;
};
```

#### 析构函数与多重继承

-   派生类的析构函数只负责清除派生类本身分配的资源，派生类的成员及基类都是自动销毁的
-   析构函数的调用顺序正好与构造函数相反

#### 多重继承的派生类的拷贝与移动操作

-   只有当派生类只用的是合成版本的拷贝、移动或赋值成员时，才会自动对其基类部分执行这些操作，否则，自定义的拷贝、赋值构造函数和赋值运算符，必须在完整的对象上执行拷贝、移动或赋值操作

### 18.3.2 类型转换与多个基类

-   在只有一个基类的情况下，派生类的指针或引用能自动转换成一个可访问的指针或引用。多个基类的情况与之类似
-   可以令某个可访问基类的指针或引用直接指向一个派生类对象

```c
void print(const Bear&);

Panda ying_yang("ying_yang");
print(ying_yang);
```

-   编译器不会在派生类向基类的几种转换中进行比较和选择，因为在它看来转换到任意基类都一样好

```c
void print(const Bear&);
void print(const Endangered&);

Panda ying_yang("ying_yang");
print(ying_yang); //二义性错误
```

### 18.3.3 多重继承下的类作用域

-   在只有一个基类的情况下，派生类的作用域嵌套在直接基类和间接基类的作用域中。查找过程沿着体系自底向上进行，直到找到所需要的名字。派生类的名字将隐藏基类的同名成员
-   在多重继承的情况下，相同的查找过程在**所有直接基类**中同时进行。如果名字在多个基类中被找到，则对该名字的使用将具有二义性
-   从它的几个基类中分别继承名字相同的成员是合法的，只不过在使用这个名字时必须明确它的版本，不加前缀限定符直接使用该名字将引发二义性
-   有时即使派生类继承的两个函数形参列表不同也可能发生错误
-   即使成员变量在一个基类中是私有的，在另一个基类中是公有的或者受保护的也可能发生错误

### 18.3.4 虚继承

-   派生类可以多次继承同一个类，派生类可以通过：
    -   它的两个直接基类分别继承同一个间接基类
    -   直接继承某个基类A，然后通过另一个类B再间接继承该类A
-   虚继承的机制解决了基类多份拷贝的问题。虚继承的目的是令某个类做出声明，承诺愿意共享它的基类
-   共享的基类子对象称为**虚基类**
-   无论虚基类在继承体系中出现了多少次，在派生类中都只包含有唯一一个共享的虚基类子对象
-   虚派生只影响从指定了虚基类A的派生类B中进一步派生出的类C，虚派生不会影响派生类B本身

#### 使用虚基类

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/img_18_3.png)

```c
//关键字public和virtual的顺序随意
class Raccoon : public virtual ZooAnimal {};
class Bear: virtual public ZoonAnimal {};

class Panda : public Bear, public  Raccoon, public Endangered {};
```

-   virtual说明符表明了一种愿望，即在后续的派生类当中共享虚基类的同一份实例。至于什么样的类能够作为虚基类并没有特殊规定
-   在该例中，Panda只有一个ZooAnimal基类部分

#### 支持向基类的常规类型转换

-   无论基类是不是虚基类，派生类对象都能被可访问基类的指针或引用操作

#### 虚基类成员的可见性

-   因为在每个共享的虚基类中只有唯一一个共享的子对象，所以该基类的成员可以被直接访问，并且不会产生二义性
-   假设类B定义了一个名为x的成员，D1和D2都是从B虚继承得到的，D继续继承了D1和D2，则在D的作用域中，x通过D的两个基类都是可见的。如果我们通过D使用x，有三种可能：
    -   如果D1和D2中没有x的定义，则x将被解析为B的成员，此时不存在二义性
    -   如果x是B的成员，同时是D1和D2中某一个的成员，则同样没有二义性，派生类的x比共享虚基类B的x的优先级高
    -   如果D1和D2中都有x的定义，则直接访问x将产生二义性问题

### 18.3.5 构造函数与虚继承

-   在虚派生中，虚基类是由最低层的派生类初始化的，以上面的例子说，由Panda的构造函数独自控制ZooAnimal的初始化过程。
-   位于派生类的最低层的类初始化共享的基类部分，即使这个低层类不是该共享基类的直接基类，低层类的构造函数也是可以在构造函数的初始化列表中用共享基类的构造函数初始化共享基类的成员

#### 虚继承的对象的构造方式

-   含有虚基类的对象的构造顺序与一般的顺序稍有不同
    -   首先，使用提供给最低层派生类构造函数的初始值初始化该对象的**虚基类**子部分   
    -   接着，按照**直接基类**在派生列表中出现的次序一次对其进行初始化
    -   最后，构造**派生类**自己

#### 构造函数与析构函数的次序

-   一个类可以有多个虚基类。此时，这些虚的子对象按照它们在派生类列表中出现的顺序依次从左向右依次构造

```c
class Character {/*...*/};

class BookCharacter : public Character {/*...*/};

class ToyAnimal {/*...*/};

class TeddyBear : public BookCharacter, public Bear, virtual public ToyAnimal
{/*...*/};
```

-   编译器按照直接基类的声明顺序对其依次进行检查，以确定其中是否含有虚基类。如果有，则先构造虚基类，然后按照声明的顺序逐一构造其它非虚基类
-   所以，该例子需要按照如下次序调用这些构造函数

```c
ZooAnimal();            //Bear的虚基类
ToyAnimal();            //直接虚基类
Character();             //第一个非虚基类的间接基类
BookCharacter();    //第一个直接非虚基类
Bear();                      //第二个直接非虚基类
TeddyBear();            //最低层派生类
```

---

-   合成的拷贝、移动构造函数、合成的赋值运算符中的成员赋值都按照例子中相同的顺序执行
-   销毁顺序与构造顺序正好相反

---

# 第19章 特殊工具与技术

## 19.1 控制内存分配

```c
//new表达式
string *sp = new string("a value");
string *arr = new string[10];

//delete表达式
delete sp;
delete [] arr;
```

-   当我们使用一条new表达式时，实际执行了三步操作
    -   new表达式调用一个名为operator new ( 或者operator new[] ) 的标准库函数。该函数分配一块足够大的，原始的，未命名的内存空间以便存储特定类型的对象（或对象的数组）
    -   编译器运行相应的构造函数以构造这些对象，并为其传入初始值
    -   对象被分配了空间并构造完成，返回一个指向该对象的指针
-   当我们使用一条delete表达式删除一个动态分配的对象时，实际执行了两步操作
    -   对指针所指的对象或者数组名所指的数组中的元素执行对应的析构函数
    -   编译器调用名为operator delete ( 或者operator delete[ ] ) 的标准库函数释放内存空间
-   如果应用程序希望控制内存分配的过程，则它们需要定义自己的operator new函数和operator delete函数
-   应用程序可以在**全局作用域**中定义operator new函数和operator delete函数，也可以将它们定义为成员函数
-   operator new或operator delete查找过程：
    -   如果被分配(释放)的对象是类类型，首先在**类及其基类的作用域**中查找
    -   **全局作用域**查找
    -   使用**标准库**定义的版本

#### operator new接口和operator delete接口

-   标准库定义了operatornew函数和operator delete函数的8个重载版本。其中前4个版本可能抛出bad_alloc异常，后4个版本则不会抛出异常

```c
//这些版本可能抛出异常
void* operator new(size_t); // 分配一个对象  
void* operator new[] (size_t); // 分配一个数组  
void* operator delete(void*) noexcept; // 释放一个对象  
void* operator delete[] (void*) noexcept; //释放一个数组 
```

```c
//这些版本承诺不会抛出异常
void* operator new(size_t, nothrow_t&) noexcept;  
void* operator new[](size_t, nothrow_t&) noexcept;  
void* operator delete(void*, nothrow_t&) noexcept;  
void* operator delete[] (void*, nothrow_t&) noexcept;  
```

-   类型nothrow_t是定义在new头文件中的一个struct
-   new头文件定义了一个名为nothrow的const对象，用户可以通过这个对象请求new的非抛出版本
-   当我们重载这些运算符时，必须使用noexcept异常说明符指定其不抛出异常
-   应用程序可以自定义上面函数版本中的任意一个，**前提**是自定义的版本必须位于**全局作用域**或者**类作用域**中
-   上述运算符函数定义成类的成员时，它们是**隐式静态**的。无需显示地声明static。且必须是静态的，而且它们不能操作类的任何数据成员

---

**operator new和operator new[]**

-   返回类型必须是void*
-   一个形参的类型必须是size_t且该形参不能含有默认实参
-   operator new函数，则可以为它提供额外的形参，使用**new的定位形式**将实参传给新增的形参
-   下面这个函数却无论如何不能被用户重载

```c
void* operator new(size_t, void*); 
```

---

**operator delete和operator delete[]**

-   返回类型必须是void
-   第一个形参的类型必须是void*
-   可以包含另外一个类型为size_t的形参，该形参的初始值是第一个形参所指对象的字节数
-   

---

-   我们提供新的operator new函数和operator delete函数的**目的在于改变内存分配的方式**，但是不管怎样，我们都不能改变new运算符和delete运算符的基本含义

---

#### malloc函数与free函数

-   可以使用malloc和free来实现operator new和operator delete
-   C++从C中继承而来，定义在cstdlib头文件

```c
void *operator new(size_t size) {
    if(void *mem = malloc(size))
        return mem;
    else
        throw bad_alloc();
}

void operator delete(void *mem) noexcept {return free(mem);}
```

### 19.1.2 定位new表达式

-   定位new与alloator的construct相似
-   对于使用operator new分配的内存空间，使用new的**定位new形式**构造对象
-   new的这种形式为分配函数提供了额外的信息。我们可以使用定位new传递一个地址，此时定位new的形式如下所示:

```c
new (place_address) type  
new (place_address) type (initializers)  
new (place_address) type [size]  
new (place_address) type [size] { braced initializer list }  
```

-   place_address必须是一个指针
-   initializers中提供一个(可能为空的)以逗号分隔的初始值列表，该初始值列表将用于构造新分配的对象
-    当仅通过一个地址值调用时，定位new使用`operator new(size_t, void*)`“分配”它的内存，该函数不分配任何内存，它只是简单地返回指针实参，然后由new表达式负责在指定的地址初始化对象以完成整个工作
-    定位new允许我们在一个特定的、预先分配的内存地址上构造对象
-    当只传入一个指针类型的实参时，定位new表达式构造对象但是不分配内存
-    但传给定位new的指针无需指向operator new分配的内存，甚至不需要指向动态内存

#### 显示的析构函数调用

-   对析构函数的显示调用与使用destroy很类似
-   可以通过对象、对象的指针或引用调用析构函数

```c
string* sp = new string("a value"); // 分配并初始化一个string对象  
sp->~string() 
```

-   调用析构函数会销毁对象，但是不会释放内存，可以重新使用该内存空间

---

## 19.2 运行时类型识别

-   运行时类型识别（run-time type identification,RTTI）的功能由两个运算符实现
    -   dynamic_cast运算符，用于将基类的指针或引用安全地转换成派生类的指针或引用
    -   typeid运算符，用于返回表达式的类型
-   当这两个运算符用于某种类型的指针或引用，且该类型含有虚函数时，运算符将使用指针或引用绑定对象的动态类型
-   适用于以下情况：想使用基类对象的指针或引用执行某个派生类操作并且该操作不是虚函数

### 19.2.1 dynamic_cast运算符

-   dynamic_cast运算符的使用形式如下所示：

```c
//e必须是一个有效的指针
dynamic_cast<type*>(e);
//e必须是一个左值
dynamic_cast<type&>(e);
//e不能是左值
dynamic_cast<type&&>(e);
```

-   type必须是一个类类型，通常情况下该类型应该含有虚函数
-   e的类型必须符合以下三个条件中的任意一个：
    -   e的类型是目标type的公有派生类
    -   e的类型是目标type的公有基类
    -   e的类型就是目标type的类型
-   失败时的结果
    -   如果一条dynamic_cast语句的转换目标是指针类型并且失败了，则结果为0
    -   如果转换目标是引用类型并且失败了，则`dynamic_cast`运算符将抛出一个`bad_cast`异常，定义在typeinfo标准库头文件中

#### 指针类型的dynamic_cast

-   假设Base类至少含有一个虚函数，Derived是Base的公有派生类
-   一个指向Base的指针bp

```c
if(Derived *dp = dynamic_cast<Derived*>(bp))
{
    //使用dp指向的Derived对象
}else
{
    //if语句的条件失败，else子句执行相应的Base操作
    //使用bp指向的Base对象
}
```

-   我们可以对一个空指针执行dynamic_cast，结果是所需类型的空指针

#### 引用类型的dynamic_cast

```c
void f(const Base &b)
{
    try{
        const Derived &d = dynamic_cast<Derived&>(b);
        //使用b引用的Derived对象
    }catch(bad_cast)
    {
        //处理类型转换失败的情况
    }
}
```

### 19.2.2 typeid运算符

-   typeid运算符：获取对象是什么类型

```c
typeid (e);
```

-   e可以是任意表达式或类型的名字
-   结果是一个常量对象的引用，该对象的类型是`type_info`或`type_info`的共有派生类型，`type_info`定义在头文件typeinfo中
-   typeid忽略顶层引用：e是一个引用，则返回该引用绑定的对象的类型，指针类似
-   typeid作用于数组，不会执行向指针的转换，获得数组的类型
-   当运算对象不属于类类型或者是一个不包含任何虚函数的类时，typeid运算符指示的是对象的**静态类型**
-   当运算对象至少定义了一个虚函数类的左值时，typeid的结果直到**运行时**才会求得

#### 使用typeid运算符

-   typeid应该作用于对象，因此我们使用*bp而非bp

```c
//这里比较的是Base*和Derived，永远不会相等
if (typeid(bp) == typeid(Derived))
{
    // bp实际指向Derived对象
}
```

-   当typeid作用于指针时（而非指针所指的对象），返回结果是该指针的静态编译类型，例如typeid(bp)的结果为Base*

---

-   只有当类类型含有虚函数时，typeid才会对表达式求值，因此，如果typeid(*p)，p指向的类型不含有虚函数，则p不必非得是一个有效的指针
-   若p有虚函数，在运行时求值，p是一个空指针，则抛出名为bad_typeid的异常

### 19.2.4 type_info类

-   type_info类必须定义在typeinfo头文件中

![](https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-20-cpp-primer-summary/tb_19_1.png)

-   type_info类一般作为一个基类出现
-   创建type_info的对象的唯一途径是使用typeid运算符
-   name的返回值因编译器而异且不一定与在程序中使用的名字一致

---

## 19.3 枚举类型

-   枚举类型使我们可以将一组整型常量组织在一起
-   每个枚举类型定义了一个新的类型。枚举属于字面常量类型
-   枚举类型分为两类：
    -   限定作用域的枚举类型
    -   不限定作用域的枚举类型

---

**定义限定作用域的枚举类型：**

```c
enum class/struct name [:type]{逗号分隔的枚举成员列表};

enum class open_modes {intput,output,append};
```

---

**定义不限定作用域的枚举类型：**

```c
enum [name] [:type]{逗号分隔的枚举成员列表};

enum color {red,yellow,green};
```

-   名字时可选的
-   如果是未命名的，则我们需要在定义该enum时定义它的对象

---

#### 枚举成员

-   在限定作用域的枚举类型中，枚举类型的作用域外是不可访问的
-   不限定作用域的枚举类型中，枚举成员的作用域与枚举类型本身的作用域相同

```c
enum color{red,yellow,green};       //不限定作用域枚举类型
enum stoplight{red,yellow,green};   //错误，重复定义枚举成员
enum class peppers{red,yellow,green};//正确，限定作用域的成员隐藏了不限定作用域的枚举成员
color eyes=green;       //能正确访问
//错误，pepper的枚举成员不在有效的作用域中
//color::green在有效作用域中，但类型错误
peppers p=green;        
color hair = color::red; //正确，允许显示访问枚举成员
peppers p2 = peppers::red //正确
```

-   默认情况下，枚举值从0开始，依次加1。也能为一个或几个枚举成员指定专门的值
-   枚举值不一定唯一，如果没有显示提供初始值，则当前枚举成员的值等于之前枚举成员的值加1
-   枚举成员是const的，因此初始化枚举成员的初始值必须是常量表达式，所以枚举成员本身也是一条常量表达式
-   可以将枚举作为：
    -   switch语句的表达式，枚举值可以用在case的标签
    -   将枚举类型作为一个非类型模板参数形参使用
    -   在类的定义中初始化枚举类型的静态成员

#### 和类一样，枚举也定义新的类型

-   要想初始化enmu对象或者为enum对象赋值，必须使用该类型的一个枚举成员或该类型的另一个对象

```c
open_modes om = 2; //错误，2不属于open_modes
om = open_modes::input; //正确
```

-   一个不限定作用域的枚举类型对象或枚举成员自动转化成整性

```c
int i = color::red; //正确，不限定作用域的枚举类型的枚举成员隐式地转换为int
int j = peppers::red; //错误，限定作用域的枚举成员不会进行隐式转换
```

#### 指定enum的大小

-   在enum的名字后加上冒号以及想要该enum中使用的类型

```c
enum intValues : unsigned long long {
    charType = 255, shortType = 65535
}
```

-   默认情况下限定作用域的enum成员类型是int。不限定作用域的枚举类型，其枚举成员不存在默认类型

#### 枚举类型的前置声明

-   可以提前声明enum，但必须指定其成员的大小

```c
//不限定作用域的枚举类型intValues的前置声明
//不限定作用域的，必须指定成员类型
enum intvalues : unsigned long long; 
//限定作用域的枚举类型可以使用默认成员类型int
enum class open_modes;
```

-   enum的声明与定义必须匹配
-   不能再同一个文件上下文中先声明一个不限定作用域的enum名字，再声明一个同名的限定作用域的enum

#### 形参匹配和枚举类型

```c
//不限定作用域的枚举类型，潜在类型因机器而异
enum Tokens { INLINE = 128, VIRTUAL = 129 };
void ff(Tokens);
void ff(int);

int main(){
    Tokens curTok = INLINE;
    ff(128);   //精确匹配ff(int)
    ff(INLINE);   //精确匹配ff(Tokens)
    ff(curTok);  //精确匹配ff(Tokens)
    return 0;
}
```

-   不能直接将整型值传给enum形参，但可以将一个不限定作用域的枚举类型的对象或枚举成员传递给整型形参。此时，enum的值提升成int或更大的整数，实际提升的结果由枚举类型的潜在类型决定

```c
void newf(unsigned char);
void newf(int);
unsigned char uc = VIRTUAL;
newf(VIRTUAL);  //调用newf(int)
newf(uc);  //调用newf(unsigned char)
```

---

#### 19.4 类成员指针

-   **成员指针**是指可以指向类的**非静态成员**的指针   
-   类的静态成员不属于任何对象，因此无须特殊的指向静态成员的指针，指向**静态成员**的指针与普通指针没有什么区别
-   成员指针的类型囊括了**类的类型**以及**成员的类型**
-   当初始化一个这样的指针时，我们令其指向类的某个成员，但是不指定该成员所属的对象
-   直到使用成员指针时，才提供成员所属的对象

### 19.4.1 数据成员指针

```c
//声明
//panda可以指向一个常量（非常量）Screen对象的string成员
//一个指向Screen类的const string成员的指针
const string Screen::*pdata;
//初始化，需要指定它所指向的成员，即非特定Screen对象的contents成员
//取址运算符作用于Screen类的成员而非内存中的一个该类对象
pdata = &Screen::contents;
//声明并初始化
//也可以使用decltype
auto pdata = &Screen::contents;
```

#### 使用数据成员指针

-   当我们初始化一个成员指针或为成员指针赋值时，该指针并没有指向任何数据
-   成员指针指定了成员而非该成员所属的对象，只有当解引用成员指针时我们才提供对象的信息
-   两种成员指针访问运算符`.*`和`->*`
-   这些运算符执行两步操作：
    -   首先，解引用成员指针以获得所需要的成员
    -   然后，像成员访问运算符一样，通过对象或指针获取成员

```c
Screen myScreen, *pScreen = &myScreen;
// .*解引用pdata以及获得myScreen对象的contents成员
auto s = myScrren.*pdata;
// ->*解引用pdata已获得pScreen所指对象的contents成员
s = pScreen->*pdata;
```

#### 返回数据成员指针的函数


```c
class Screen {
public:
    //data是一个静态成员，返回一个成员指针
    static const std::string Screen::* data() 
	    { return &Screen::contents; }
}

//data()返回一个指向Screen类的contents成员的指针
const string Screen::*pdata = Screen::data();

//获得myScreen对象的contents成员
auto s = myScreen.*pdata;
```

### 19.4.2 成员函数指针

-   可以定义指向类的成员函数的指针

```c
class Screen {
    char get_cursor() const { return contents[cursor]; }
    inline char get(pos ht, pos wd) const;
}

//pmf是一个指针，它可以指向Screen的某个常量成员函数
//前提是该函数不接受任何实参，并且返回一个char
auto pmf = &Screen::get_cursor;

//*pm2两端的括号必不可少
char (Screen::*pm2)(Screen::pos,Screen::pos) const;
pmf2 = &Screen::get;
```

-   成员函数和指向该成员的指针直线不存在自动类型转换规则：

```c
//必须显示地使用取址运算符
pmf = &Screen::get;
//错误，在成员函数和指针之间不存在自动转换规则
pmf = Screen::get;
```

#### 使用成员函数指针

-   使用`.*`或者`->*`运算符作用于指向成员函数的指针，以调用类的成员函数

```c
Screen myScreen, *pScreen = &myScreen;
//通过pScreen所指的对象调用pmf所指的函数，括号必不可少
char c1 = (pScreen->*pmf)();
//通过myScreen对象将实参0,0传给含有两个形参的get函数
char c2 = (myScreen.*pmf2)(0,0);
```

#### 使用成员指针的类型别名

```c
//Action是一种可以指向Screen成员函数的指针
using Action = char (Screen::*)(Screen::pos,Screen::pos) const;

Action get = &Screen::get;
```

-   可以将指向成员函数的指针作为某个函数的返回类型或形参类型，指向成员的指针形参也可以拥有默认实参

```c
//action接受一个Screen的引用，和一个指向Screen成员函数的指针
Screen& action(Screen&, Action = &Screen::get);

Screen MyScreen;
auto get = &MyScreen::get;
//等价的调用
action(MyScreen); //使用默认实参
action(MyScreen,get); //使用之前定义的变量get
aciton(MyScreen,&MyScreen::get);
```

### 19.4.3 将成员函数用作可调用对象

-   与普通函数不同，成员指针不是可调用对象，这样的指针不支持函数调用运算符，所以我们不能直接将一个指向成员函数的指针传递给算法

```c
//fp指向string的empty函数
auto fp = &string::empty;
//错误，必须使用.*或者->*调用成员函数
find_if(svec.begin(),svec.end(),fp);

//上面的句子表示，尝试在find_if内部通过以下方式来调用fp
//这是错误的，要想通过成员指针调用函数，必须使用->*或.*运算符
//正确的使用应该是：if( ((*it).*fp)() )
if(fp(*it));
```

-   想要把成员函数转换为可调用调用，有以下方法：
    -   使用function
    -   使用mem_fn
    -   使用bind

#### 使用function生成一个可调用对象

-   从指向成员函数的指针获取可调用对象的一种方法是使用标准库模板function

```c
//告诉function，emptye接受一个string参数，并返回bool
//执行成员函数的对象将被传给隐式的this形参，即将this传递给const string&形参
function<bool (const string&)> fcn = &string::empty;
find_if(svec.begin(),svec.end(),fcn);

//相当于在find_if内部，使用一下方式来调用fcn
//假设it是find_it内部的迭代器，则*it是给定范围内的一个对象
if(fcn(*it))

//function类将函数调用转换为如下形式：
//假设p是fcn内部的一个指向成员函数的指针
if( ((*it).*p)() )
```

-   当我们定义一个function对象时，必须指定该对象所能表示的函数类型，即可调用对象的形式
-   如果可调用对象是一个成员函数，则第一个形参必须表示该成员是哪个（一般是隐式的）对象上执行的（如上面的const string&）
-   我们提供给function的形式中还必须指明对象是否是以**指针**（使用`->*`调用）或**引用**（使用`.*`调用）的形式传入的

```c
vector<string*> pvec;
function<bool (const string*)> fp = &string::empty;
//fp接受一个指向string的指针，然后调用->*emptye
find_if(pvec.begin(),pvec.end(),fp);
```

#### 使用mem_fn生成一个可调用对象

-   mem_fn定义在functional头文件中
-   mem_fn让编译器负责推断成员的类型，并且可以从成员指针生成一个可调用对象
-   mem_fn可以根据成员指针的类型推断可调用对象的类型，而无须用户显式地指定

```c
find_if(svec.begin(),svec.end(),mem_fn(&string::empty));
```

-   `mem_fn(&string::empty)`生成一个可调用对象，该对象接受一个string实参，返回一个bool值

-   `mem_fn`生成的可调用对象可以通过**对象调用**，也可以通过**指针调用**
-   可以认为，`mem_fn`生成的可调用对象含有一对重载的函数运算符：一个接受string*，另一个接受string&

```c
auto f = mem_fn(&string::empty); //f接受一个string或者一个string*
f(*svec.begin()); //正确，传入一个string对象，f调用.*调用empty
f(&svec[0]); //正确，传入一个string的指针，f调用->*调用empty
```

#### 使用bind生成一个可调用对象

```c
//选择范围内的每个string，并将其bind到empty的第一个隐式实参上
auto it = find_if(sevc.begin(),svec.end(),bind(&string::empty,_1));
```

-   当我们使用bind时，必须将函数中用于表示知晓对象的隐式形参转换为显示的
-   bind生成的可调用对象的第一个实参既可以是string的指针，也可以是string的引用

```c
auto f = bind(&string::empty,_1);
f(*svec.begin()); //正确，实参是一个string,f使用.*调用empty 
f(&svec[0]); //正确，实参是一个string的指针，f使用->*调用empty
```

---

## 19.5 嵌套类

-   一个类可以定义在另一个类的内部，前者称为嵌套类或嵌套类型
-   嵌套类常用于定义作用实现部分的类
-   嵌套类是一个独立的类，与外层类基本没什么关系
-   外层类的对象和嵌套类的对象是相互独立的
-   嵌套**类的名字**在外层类作用域中是可见的，在外层类作用域之外不可见
-   嵌套**类的名字**不会和别的作用域中的同一名字冲突
-   外层类对嵌套类的成员没有特殊的访问权限，同样，嵌套类对外层类的成员也没有特殊的访问权限
-   嵌套类在其外层类中定义了一个类型成员，该类型的访问权限由外层类决定（位于外层类的public、private、protected相应的位置就有相应权限）

#### 声明一个嵌套类

```c
class TextQuery {
public:
    class QueryResult;  //嵌套类稍后定义
};
```

#### 在外层类之外定义一个嵌套类

-   嵌套类必须**声明**在类的内部，但是可以**定义**在类的内部或外部
-   在嵌套类在其外层类之外完成真正的定义之前，它都是一个不完全类型

```c
//定义嵌套类
class TextQuery::QueryResult {
    //...
};
```

#### 定义嵌套类的成员

```c
//在外层类之外定义嵌套类的构造函数
TextQuery::QueryResult::QueryResult(//...){}
```
#### 嵌套类的静态成员定义

-   如果QueryResult（嵌套类）**声明了**（在类内）一个静态成员，则该成员的定义将位于TextQuery（外层类）的作用域之外

```c
int TextQuery::QueryResult::static_mem = 1024;
```

#### 嵌套类作用域中的名字查找

-   外层类的成员可以像使用任何其他类型成员一样使用嵌套类的名字
-   因为QueryResult嵌套在TextQuery中，所以TextQuery的query成员可以直接使用名字QueryResult
 
#### 嵌套类和外层类是相互独立的

-   外层类的对象和嵌套类的对象没有任何关系
-   嵌套类的对象只包含嵌套类定义的成员
-   外层类的对象只包含外层类定义的成员，在外层类对象中不会有任何嵌套类的成员

#### union：一种节省空间的类

-   联合（union）是一种特殊的类，一个union定义了一种新类型
-   一个union可以有多个数据成员，但是在任意时刻只有一个数据成员可以有值
-   当我们给某个成员赋值之后，该union的其他成员就变成未定义状态的了
-   union的存储空间至少要容纳它的最大的数据成员
-   union不能含有引用类型的成员
-   union可以为其成员指定public、private、protected，默认情况下为公有的
-   union可以定义包括构造函数和析构函数在内的成员函数
-   由于union不能继承其他类，也不能作为基类使用，所以union不能含有虚函数

#### 定义union

-   union提供了一种有效的途径是的我们可以方便地表示一组类型不同的**互斥值**

```c
union Token {
    char cval;
    int ival;
    double dval;
};
```

#### 使用union类型

```c
Token first_token = {`a`}; //初始化cval成员
Token last_token; //未初始化的Token对象
Token *pt = new Token; //指向一个未初始化的Token对象的指针
```

-   如果union提供了初始值，该初始值用于初始化**第一个**成员

-   使用通用的成员访问运算符访问一个union对象的成员

```c
last_token.calv = 'z';
pt->ival = 42;
```

#### 匿名union

-   匿名union是一个未命名的union
-   一旦定义了一个匿名union，编译器就自动地位该union创建一个未命名的对象
-   在匿名union的定义所在的作用域内该union的成员都是可以直接访问的
-   匿名union不能包含受保护的成员或私有成员，也不能定义成员函数

```c
union {
    char cval;
    int ival;
    double dval;
};

cval = 'c';
ival = 42;
```

#### 含有类类型的union

-   C++早起版本，union不能含有定义了构造函数或拷贝控制成员的类类型，C++11取消了这一限制
-   有含有定义了构造函数或拷贝控制成员的类类型的成员的union，比只含有内置类型成员的union复杂得多
-   想将union的值改为类类型成员对应的值，或将类类型成员的值改为其他值，必须分别构造或析构该类类型的成员
-   当union包含的是内置类型的成员时，编译器将按照成员的次序依次合成union的默认构造函数或拷贝控制成员。但是如果union含有类类型的成员，并且该类型自定义了默认构造函数或拷贝控制成员，则编译器将为union合成对应的版本并将其声明为删除的
    -   **例如：**
    -   string类定义了五个拷贝控制成员以及一个默认构造函数
    -   如果union含有string类型的成员，并且union没有自定义默认构造函数或某个拷贝控制成员，则编译器将合成缺少的拷贝控制成员并声明成删除的
    -   如果在某个类中含有一个union成员，而且该union含有删除的拷贝控制成员，则该类与之对应的拷贝控制操作也将是删除的

#### 使用类管理union成员

-   对于union来说，要想构造或销毁类类型的成员必须执行非常复杂的操作，因此我们通常把含有类类型成员的union内嵌在另一个类当中。这个类可以管理并控制与union的类类型成员有关的状态转换
-   为了追踪union中到底存储了什么类型的值，我们通常会定义一个独立的对象，该对象称为union的判别式，通常用enum类型表示

---

-   如果union当前存储的不是string，则我们找不到一个已存在的string对象提供我们调用运算符。此时，我们必须：
    -   1.先利用定位new表达式在内存中为sval构造一个string  
    -   2.然后将该string初始化string形参的副本
    -   3.最后更新判别式的返回结果

```c
class Token {
union {
    char cval;
    int ival;
    double dval;
    std::string sval;
};

enum {INT,CHAR,DBL,STR} tok;
}；

//第1、2步
new(&sval) string(s);
//第3步
tok = STR;
```

---

## 19.7 局部类

-   类可以定义在某个函数的内部，称为局部类
-   局部类定义的类型只能在它的作用域内可见
-   和嵌套类不同，局部类的成员受到严格限制
-   局部类的所有成员（包括函数在内）都必须完整定义在类的内部
-   局部类中不允许声明静态数据成员

#### 局部类不能使用函数作用域中的变量

-   局部类只能访问外层作用域定义的：
    -   类型名
    -   静态变量
    -   枚举成员
-   局部类如果定义在某个函数内部，则不能访问该函数的普通局部变量

```c
int a,val;
void foo(int val)
{
    static int si;
    enum Loc {a=1024,b};
    //Bar是foo的局部类
    struct Bar{
        Loc LocVal; //正确，使用一个局部类型名
        int barVal;
        
        void fooBar(Loc l = a) //正确，默认实参是Loc::a
        {
            barVal = val; //错误，val是foo的局部变量
            barVal = ::val; //正确，使用一个全局对象
            varVal = si; //正确，使用一个静态对象
            localVal = b; //正确，使用一个枚举成员
        }
    };
}
```

#### 常规的访问保护规则对局部类同样适用

-   外层函数对局部类的私有成员没有任何访问权限
-   局部类可以将外层函数声明为友元

#### 嵌套的局部类

-   可以在局部类的内部再嵌套一个类，此时，嵌套类的定义可以出现在局部类之外。不过，嵌套类必须定义在于局部类相同的作用域中
-   局部类的嵌套类也是一个局部类，必须遵循局部类的各种规定

---

## 19.8 固有的不可移植的特性

-   为了支持低层编程，C++定义了一些固有的不可移植的特性
-   所谓不可移植的特性是指因**机器而异**的特性，当我们将含有不可移植的程序从一台机器转移到另一台机器上时，通常需要重新编写该程序，例如：
    -   算术类型的大小在不同机器上不一样

### 19.8.1 位域

-   类可以将其（非静态）数据成员定义成位域（bit-field）
-   在一个位域中含有一定数量的二进制位
-   当一个程序需要向其他程序或硬件设备传递二进制数据时，通常会用到位域
-   位域在内存中的布局是与机器相关的
-   位域的类型必须是整型或枚举类型，通常情况下我们使用无符号类保存一个位域

```c
typedef unsigned int Bit;
class File{
    Bit mode: 2; //mode占2位
    Bit modified: 1; 
    Bit prot_owner: 3;
    Bit prot_group: 3;
    Bit prot_world: 3;
    //...
public :
    //文件类型以八进制的形式表示
    enum modes{ READ = 01, WRITE = 02, EXCUTE = 03};
}
```

-   如果可能的话，在类的内部连续定义的位域压缩在同一个整数的相邻位，从而提供存储压缩。如上例中，五个位域可能会存储在同一个unsigned int中
-   这些二进制**能否压缩**到一个整数中以及**如何压缩**与**机器相关**
-   取址符不能作用域位域，因此任何指针都无法提供向指向类的位域

#### 使用位域

```c
modified = 1;
```

-   通常使用内置的位运算符操作超过1位的位域

```c
mode |= READ;
if (mode & WRITE)
```

### 19.8.2 volatile限定符

-   volatile的确切含义与机器有关，只能通过阅读编译器文档来理解
-   程序可能包含一个由系统时钟定时更新的变量。当对象的值可能在程序的控制或检测之外被改变时，应该将该对象声明为volatile。关键字volatile告诉编译器不应对这样的对象进行优化

```c
volatile int display_register; //该int值可能发生改变
volatile Task *curr_task;  //curr_task指向一个volatile对象
volatile int iax[max_size]; //iax的每个元素都是volatile
volatile Screen bitmapBuf; //bitmapBuf的每个成员都是volatile
```

-   可以将成员函数定义为volatile，只有volatile的成员函数才能被volatile的对象调用

---

-   可以声明volatile指针、指向volatile对象的指针、指向volatile对象的volatile指针
-   只能将一个volatile对象的地址（或者拷贝一个指向volatile类型的指针）赋给一个指向volatile的指针
-   只有当某个引用时volatile的时，我们才能使用一个volatile对象初始化该引用

```c
volatile int v; //v是一个volatile int
int *volatile vip; //vip是一个volatile指针，它指向int 
volatile int *ivp; //ivp是一个指针，它指向一个volatile int
volatile int *volatile vivp; //vivp是一个volatile指针，它指向volatile int

int *ip = &v; //错误，必须使用指向volatile int的指针
ivp = &v; //正确
vivp = &v; //正确
```

#### 合成的拷贝对volatile对象无效

-   不能使用合成的拷贝/移动构造函数及赋值运算符初始化volatile对象或从volatile对象赋值。合成的成员接受的形参类型是（非volatile）常量引用，显然我们不能把一个非volatile引用绑定到一个volatile对象上
-   必须之定义拷贝或移动操作

```c
class Foo{
public:
    Foo(const volatile Foo&);
    Foo& operator=(volatile const Foo&);
    Foo& operator=(volatile const Foo&) volatile; //只能由volatile对象调用
}
```

### 19.8. 3 链接指示：extern “C”

-   对于其他语言编写的函数来说，编译器检查其调用的方式与处理普通C++函数的方式相同，但是生成的代码有所区别
-   C++使用链接指示指出任意非C++函数所用的语言
-   要想把C++代码和其他语言（包括C语言）编写的代码放在一起，要求我们必须有权访问该语言的编译器，并且这个编译器与当前的C++编译器是兼容的

#### 声明一个非C++得函数

-   链接指示可以有两种形式：
    -   单个的
    -   复合的
-   链接指示不能出现在**类定义**或**函数定义**的内部
-   同样的链接指示必须在函数的每个声明中都出现

```c
//可能出现在C++头文件<cstring>中的链接指示

// 单语句链接指示
extern "C" size_t strlen(const char*);
// 复合语句链接指示
extern "C"
{
    int strcmp(const char*, const char*);
    char* strcat(char*, const char*);
}
```

-   字符串字面值常量指出了编写函数所用的语言，如：
    -   extern "Ada"
    -   extern "FORTRAN"

#### 链接指示与头文件

-   复合语句链接指示中，花括号的作用是将适用于该链接指示的多个声明聚合在一起，否则花括号就会被忽略，花括号中声明的函数名字就是可见的，就好像在花括号之外声明的一样
-   多重声明的形式可以用于整个头文件。头文件中的所有普通函数声明都被认为是由链接指示的语言编写的。链接指示可以嵌套，因此如果头文件包含自带链接指示的函数，则该函数的链接不受影响

```c
extern "C" {
    #include <string.h> 
}
```

#### 指向extern "C"函数的指针

-   编写函数所用的语言（extern "C"）是函数类型的一部分，对于使用链接指示定义的函数来说，它的每个声明都必须使用相同的链接指示
-   指向其他语言编写的函数的函数指针必须与函数本身使用相同的链接指示

```c
// pf指向一个C函数，该函数接受一个int返回void
//当我们使用pf调用函数时，编译器认为当前调用的是一个C函数
extern "C" void (*pf)(int);

void (*pf1)(int); //指向一个C++函数
extern "C" void (*pf2)(int); //指向一个C函数
pf1 = pf2; //错误，pf1与pf2的类型不同
```

#### 链接指示对整个声明都有效

-   当我们使用链接指示时，它不仅对函数有效，而且对作为返回类型或形参类型的函数指针也有效

```c
//f1是一个C函数，它的形参是一个指向C函数的指针
//这个链接指示不仅对f1有效，对作为参数的函数指针同样有效
extern "C" void f1(void(*)(int));
```

-   如果希望给C++传入一个指向C函数的指针，则必须用类型别名

```c
//FC是一个指向C函数的指针
extern "C" typedef void FC(int);
//f2是一个C++函数，该函数的形参是指向C函数的指针
void f2(FC *);
```

#### 导出C++函数到其他语言

-   通过使用链接指示对函数进行定义，我们可以令一个C++函数在其他语言编写的程序中可用

```c
//定义了一个函数calc，该函数可以被C程序调用
//编译器将为该函数生成适合于指定语言的代码
extern "C" double calc(double dparm){//函数的实际实现}
```

-   可被多重语言共享的函数的返回类型或形参类型受到很多限制，例如：不太可能把一个C++类的对象传递给C程序，因为C程序无法理解构造函数、虚构函数以及其他类特有的操作

#### 对链接到C的预处理其的支持

-   有时需要在C和C++中编译同一个源文件，为了实现这一目的，在编译C++版本的程序时预处理其定义__cplusplus

```c
#iddef __cplusplus
//如果定义了__cplusplus，则编译为C++程序
extern "C"
#endif
int strcmp(const char*,const char*);
```

#### 重载函数与链接指示

-   链接指示与重载函数的相互作用依赖于**目标语言**。如果目标语言支持重载函数，则为该语言实现链接指示的编译器很可能也支持重载这些C++的函数

---

-   由于C语言不支持函数重载，因此有：

```c
//错误，两个extern "C"函数的名字相同
extern "C" void print(string char*);
extern "C" void print(int);
```

-   如果一组重载函数中有一个是C函数，则其余的必定是一个C++函数

```c
//C函数可以中C或C++程序中调用
//C++函数重载了该函数，可以在C++程序中调用
extern "C" double cale(double);
extern SamllInt cale(const SmallInt&);
extern BigNum cale(const BigNum&);
```

-   C版本的cale函数可以在C或C++程序中调用，而使用了类类型形参的C++函数只能在C++程序中调用




