---
layout:     post
title:      "「二十三」《C++Primber》笔记 第IV部分"
date:       2018-02-09 09:00:00
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
uniform_random_distribution<unsigned> u(0,9);
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



