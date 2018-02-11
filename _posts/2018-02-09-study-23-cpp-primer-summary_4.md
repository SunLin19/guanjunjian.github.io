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

