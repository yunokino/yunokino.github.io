---
layout: post
title:  "《Effective C++》读书笔记"
---
### 习惯C++
C++是一种多范型编程语言，主要的部分是 : C语言，面向对象，泛型编程，STL库

使用const、enum和inline代替预编译的denfine指令，因为define并不在代码中，无法被编译器感知，所以可能会出现一些错误

尽量多使用const，包括变量，函数，参数等  
成员函数的const形式不会改变class内的任何东西，然而const指针所指向的数据却可以改变(且无法更正)，所以又有mutable关键字可以移除相应数据的常量性，但是被声明为const的函数可以更改的数据依旧被要求不能被客户端(用户)使用
const和non-const成员函数可以通过对non-const进行类型转化调用const来进行代码复用

在使用对象前需要进行初始化,此处的对象包括内置类型int,long,char*等，内置类型需要注意手工完成初始化，否则可能会出现未定义的错误。
在对构造函数进行初始化时，最好使用成员初值列的形式，而不是在函数内进行assignment，因为c++在进入构造函数前会调用构造函数对对象进行初始化
> 此处的对象不包括内置类型，因为如int等没有构造函数，需要自己格外注意初始化，书中给出的建议是当内置类型的数量较多时，可以写一个函数单独进行内置类型的初始化，既不会产生单独的开销也较为美观  

这种情况下可以节省构造函数的开支。  
顺带着，成员变量的初始化顺序与class的声明顺序相同，而与调用时的无关，尽管如此也要保持一致，以免混淆。
最后一点:**不同编译单元内定义的non-local static对象的初始化顺序**  
> 编译单元指的是产出单一目标文件的源码，理解成一份源码即可  

需要调用另一份文件的static对象时，将non-local static对象搬到函数中，返回一个引用(reference)，来提供调用接口，实现non-local static-> local static的转变，因为local static对象会在被调用的时候进行初始化，示例代码如下:  
```c++
FileSysytem& tfs()
{
    static FileSystem fs;
    return fs;
}
```
通过`tsf().`便可以调用FileSystem对象 

### 构造/析构/赋值运算

C++会为class编写default构造函数，copy构造函数，copy assignment操作符重载和析构函数
> 当函数被调用(且被需要)的时候才会被创建  

copy和copy assignment编译器会将非static成员变量拷贝到目标对象  
但是假如变量中含有const,reference或者基类中有private的=重载的话，编译器会拒绝自动编写相应代码，此时可以自己处理
> 也比较好理解，毕竟const不可改变，reference不可改变指向，而派生类的=重载自然无法调用基类的private函数

拒绝调用自动生成的函数只需要将其声明为private并不去实现就好，这样的话即使是成员函数和friend调用时也会出现linkage error
> iostream中的ios_base和basic_ios和sentry中便利用了这种技巧  

另一种将错误出现前移的方法(即当member fun或者friend class调用时错误在编译时出现)是编写一个base class并编写对应的copy fun和copy assinment设置为private让相应的class继承即可，代码如下:
```c++
class Uncopyable {
    protected:
        Uncopyable(){};
        ~Uncopyable(){};
    private:
        Uncopyable(const Uncopyable&);//参数不需要名字
        Uncopyable& operator=(const Uncopyable&);
}

class test:private Uncopyable{
    // something to do
};
```
>太妙了!利用了编译器和自动生成函数的特性将派生类自动生成函数的行为拒绝，利用'错误'进行禁止！  

为多态基类声明virtual析构函数，派生类里或许会有一些基类没有释放的东西，而这很明显会造成内存泄漏，所以需要派生类定制析构函数  
>纯虚函数的声明为
```c++ 
virtual void funtion1()=0;
```  

别让异常逃离析构函数，使用try_catch捕捉异常，并且应该使用函数来处理异常。  

不要在构造和析构函数中调用virtual函数，因为基类会先调用属于自己的虚函数，而不会下降到派生类中  
事实上此时对象的类型就是base class，派生类的变量处于未定义状态  
在基类中声明一个纯虚函数并且在构造(或析构)调用会在连接期报错，因为并没有要执行的内容，但是事情并没有那么容易，一些情况下可以通过封装调用虚函数的初始化代码为函数去除编译器报错，但是此时并不是预期的版本。  
一种解决办法是将虚函数改为带参数的非虚函数，根据派生类传来的信息进行相应操作。

对operator=的重载返回值应该是一个this指针的引用，目的是实现连续赋值  
> 返回一个左值引用，而c++新特性又添加了右值引用，引入右值引用，就是为了移动语义。移动语义就是为了减少拷贝。std::move就是将左值转为右值引用。这样就可以重载到移动构造函数了，移动构造函数将指针赋值一下就好了，不用深拷贝了，提高性能 ( 资料来源[https://zhuanlan.zhihu.com/p/335994370] )

处理operator=的自我赋值 1、使用**证同测试**，在删除之前查看是否相同的元素 2、在删除某些元素之前拷贝副本，删除副本即可 3、copy和swap方式  
所有的代码的目的都是使得程序具有安全性。  

copy all parts of an object，只是一个提醒。相应的派生类需要调用基类的函数进行拷贝  
(1) 复制所有local变量 (2) 调用基类的拷贝函数  
还有不要用copy assignment调用copy构造函数，反向同样不合理
>然而在OOP作业里面却是标准答案，令人感慨  

代码复用的合理做法是声明一个私有的init函数进行处理相同部分


### 资源管理

以对象管理资源  
利用析构函数自动释放资源，自动的相比于手动总是令人更加安心  
可以使用auto_ptr来进行管理，auto_ptr是一个"类指针对象"，会自动释放内存
> auto_ptr会在被销毁之后删除所指对象，所以不能有多个auto_ptr指向同一对象，也因此复制通过构造函数或‘=’复制auto_ptr时原先的对象会被设置为null，所以在C++11后便因为出现问题后难以定位被废除，此处仅供了解  

更明智的做法是用shared_ptr，采用引用计数来进行资源回收，需要注意的是动态分配的array并不能通过shared_pt正常释放
>因为shared_ptr使用delete而不是delete[]来释放内存  

RALL观念 : 获取资源时初始化  
对RALL对象的复制需要根据情况来选择合理的处理
1. 禁止复制，继承前文所提到的Uncopyable
2. 使用引用计数法也就是shared_ptr并自定义删除器  

在资源管理类中提供对原始资源的访问
显式一般是get()等函数，隐式更加方便，但也有更大的可能出问题，隐式代码如下
```c++
class Font {
    public:
    ...
    operator FontHandle() const 
    { return f; } // f是原始资源，字体句柄
    ...
}
```

new与delete相对应，也就是如果动态申请数组那么就要delete[]，其他的用delete即可  

C++中参数的计算顺序并不固定，所以要以独立语句将new对象放入智能指针里，例如下面的代码就可能内存泄漏，尽管使用了shared_ptr
```c++
processWidget(std::tr1::shared_ptr<Widget>(new Widget),priority());
// 如果先执行new Widget再执行priority()，并且其抛出异常，那么new Widget还没有与shared_ptr绑定，会造成内存泄漏
```
解决办法：单独语句储存new对象  
```C++
std::tr1::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```


### 设计与声明
开发易于正确使用的接口  
引入新类型来进行引导，比如时间
```c++
/* 有意的引导参数类型 */
struct Day {
    explicit Day(int d)
        : val(d) {}
    int day;
}
struct Month {
    ...
}
struct Year {
    ...
}
Date(const Month &m, const Day &d, const Year &y);
/*
相较于
Date(int m, int d, int y);
上述发生错误的情况明显更少
*/
```
阻止误用的方法包括建立新类型，限制类型操作，束缚对象值，消除客户的资源管理责任  
接口的重要特性是 **一致性**，尽量使得接口与内置类型的表现一样，在此基础上利于提供同样的接口
>比如STL容器一般都有一个size接口  

shared_ptr提供接口，可以定制删除器，防范DLL问题  

定义class如同定义type，需要慎重思考关于资源管理和初始化等问题  

Prefer pass-by-referencce-to-const to pass-by-value.  
C++缺省方式以pass-by-value传递参数，调用copy构造函数完成副本构建，会有很大的开销。  
pass-by-referencce-to-const也会防止**对象切割**问题，即派生类传递到基类的参数中后属于派生类的部分消失了，只剩下了基类。
对内置类型和STL的迭代器和函数对象来说，pass-by-value更加合适。

 返回对象的时候返回对象即可，不要返回对象的引用

 成员变量需要声明为private，为了封装，并且，protected和public一样的封装性不好

 以non-member、non-friend替换member函数，原因在于应该让尽量少的函数能访问私有变量，这同样是为了封装性  
 一个比较自然的做法是将其组装成一个命名空间，而且命名空间可以跨文件，所以可以将操作不同对象的函数分离开来

 如果要为函数的所有参数进行类型转化，最好把其写成non-member函数
 ```c++
 const Rational operator*(const Rational &lhs,const Rational &rhs) {
    ...
 }
 ```
 这样可以让乘法两边都能进行类型转换  

 当swap函数不合适时，考虑自定义一个swap函数
 > 模板全特化 : 针对特定类型调用一个特定函数，参数全部被指定  
 偏特化也是为了给自定义一个参数集合的模板，但只有部分参数被指定，偏特化后的模板需要进一步的实例化才能形成确定的签名。 值得注意的是函数模板不允许偏特化，使用重载可以替代
 
 ### 实现
 尽量延后变量定义的时间，同样是为构造和析构成本的减少，甚至直到其需要赋初值的时候方才定义  

 少做转型，即使必须尽量封装或者使用新式转型  
 旧式转型 : T(expr) or (T)expr  
 新式转型 : 
 const_cast<T>(expr) : 移除常量性  
 dynamic_cast<T>(expr) : 安全向下转型  
 reinterpret_cast<T>(expr) : 低级转型  
 static_cast<T>(expr) : 强迫隐式转换  
 转型总会带来一些开支
 > derived class ptr 有时添加offset来获取 base class ptr (根据编译器)  
 
 另一种情况:
 ```c++
 class SpecialWindow : public Window{
     public :
        virtual void onResize() {
          static_cast<Window>(*this).onResize();
    ...
    }
 }
 ```
 转型时调用的是基类**副本**的虚函数再调用当前派生类的虚函数，所以会产生问题，应该改成如下:
 ```c++
 ...
 Window::onResize();
 ...
 ```
至于dynamic_cast，用来转型指向派生类的基类指针，但是在很多实现版本中速度很慢(有些版本甚至用class字符串比较)，有两种做法  
1. 类中储存指向派生类的智能指针，但问题是无法对每个派生类都进行处理
2. 在基类中声明一个什么也不做的虚函数  
   
视情况而定，能用时尽量用  

避免返回指向对象内部的句柄(引用，指针，迭代器)，一方面可以增加封装性，一方面减少dangling handles(空悬句柄)的可能
> 空悬句柄即指向的对象已经不存在的句柄  
  

异常安全性的两个条件:
  1. 不泄露任何资源
  2. 不允许数据破坏

异常安全函数有三种类型 : 基本型(即使有异常，函数依然会有效)、强烈型(函数如果无效，会回到调用前的状态)、不抛异常型(不会有异常)  
强烈保证型可以通过 copy and swap 实现，即先创造副本并在其上面操作，成功后将原对象与副本swap  
如果函数中调用了其他函数，函数的安全保证将是调用函数中的最弱者一样。  

 inline函数会将调用函数的地方替换为函数体，所以最好用在小型，频繁调用的函数上
> 类的成员函数若实现在类体内，则是一个隐式的inline申请  

需要注意的是需要对构造和析构函数谨慎使用inline，因为即使声明为空，C++也会生成成员变量的构造过程，可能造成大量代码生成  

将文件之间的编译依赖关系降到最低。  
将声明式与定义式分开为不同的文件(例如C++标准实现库有\<iosfwd\>来进行声明)  
Pimpl类将类的实现和类本身解耦，类本身存储一个实现类的智能指针，此时修改实现并不会导致引用类的多个文件的重新编译，被称为句柄函数，还有一种编写句柄函数方式是将类变成抽象基类  

### 继承和面向对象设计
public继承塑造的是一种'is-a'关系，即派生类是基类的特殊化概念，**所有在基类中允许的事在派生类中应当也有效**，但是有时也要考虑特殊情况，如以下代码:
```C++
class Bird {
    virtual void fly();
};
class Penguin : public Bird {
    ...
};
```
企鹅是鸟但不会飞，所以public继承就有问题，应该认真考虑  
&nbsp;  
避免遮蔽继承的名称，因为环境链的原因，若派生类和基类有同样名称的函数，只会使用派生类的函数，解决办法为使用using指令或者编写转交函数，即将任务传递给其他函数:
 ```c++
 virtual void mf1() {
    Base::mf1( );
 }
 ```  

&nbsp;  
区分接口继承和实现继承，类似于函数声明和函数定义  
函数分为三块 : pure-virtual impure-virtual non-virtual  
1. pure-virtual，纯虚函数，只提供接口，但事实上可以添加实现，此时可以视为必须声明使用的缺省继承
2. impure-virtual，非纯虚函数，提供接口和缺省实现
3. non-virtual，非虚函数，提供接口和强制实现
> C++虚函数是多态性实现的重要方式，当某个虚函数通过指针或者引用调用时，编译器产生的代码直到运行时才能确定调用函数的版本。被调用的函数是与绑定到指针或者引用上的对象的动态类型相匹配的那个。因此，借助虚函数，我们可以实现多态性。

</br>

考虑虚函数以外的选择。换种思路。  
1. non-virtual interface
2. 函数指针
3. tr1::function替换virtual函数
4. 将此体系的virtual函数替换为另一体系里的virtual函数

&nbsp;  
绝不重新定义继承的非虚函数。  

&nbsp;  
绝不重新定义继承而来的缺省参数值  
> 缺省参数静态绑定，virtual函数动态绑定

&nbsp;  
通过复合塑膜'has-a'(对应应用域，即现实世界中的事物)或'is-implented-in-terms-of'(对应实现域，即为了某些目的而实现的对象)
复合就是内嵌其它类型的对象。  
&nbsp;  
审慎地使用private继承。  
private继承意思与复合一样，即'is-implented-in-terms-of'，但复合一般情况下更好。private继承可以进行EBO优化，有事可以选用
> EBO优化 : 独立空类会被C++赋予char大小的内存值(为保证同一类型的不同对象地址始终有别，要求任何对象或成员子对象的大小至少为1)，当复合时由于字节对齐可能会造成更多空间浪费，而继承会对此进行优化，减少内存浪费。

在STL中EBO优化应用广泛，如vector的核心类`_Vector_impl` 继承了空间配置器`_Tp_alloc_type`  
&nbsp;  
明智而审慎的使用多重继承  
使用virtual继承可以避免'钻石型多重继承'
> 标准库中有钻石型多重继承结构basic_ios->basic_istream, basic_ios->basic_ostream, 而basic_iostream同时继承basic_istream和basic_ostream

但是虚继承成本较高，一方面，访问虚继承的基类成员变量速度较慢，而且体积更大，另一方面，基类的初始化由最远的派生类负责，这也导致一些成本。当虚基类没有任何数据时比较有实用价值。  
当public继承接口基类和private继承实现基类时，可以使用多重继承。  
### 模板和泛型编程
了解隐式接口和编译器多态。  
class支持显式接口(函数的签名式)和运行期多态(虚函数)  
template支持隐式接口和编译器多态
隐式接口   : 由一组有效表达式组成 (对参数对象施加的接口约束)，只要通过类型转换表达式成立即可
编译器多态 : 指的是在编译期template具现化模板函数调用不同的函数    
&nbsp;  
了解typename的双重含义.  
声明模板时`class`和`typename`的作用完全相同  
从属名称 : `template`内的名称依赖于`template`参数则为从属名称，如果又嵌套于模板参数中则为嵌套从属类型名称，如:
```c++ 
template<typename C>
void print2nd(const C& container) 
{
    C::const_iterator x = container.begin(); //本意声明一个迭代器
}
```
嵌套从属名称可能引起解析错误，例如假如C::const_iterator是一个变量名，语句就会变成相乘的式子。但是不能在基类列(继承的地方)和成员初值列使用`typename`.    
&nbsp;  
学习处理模板化基类内的名称。  
一般而言，C++会拒绝进入模板基类中使用函数，因为可能会有一些特化版本，如果确定使用，可以用this指针指定或者using指令来明确保证使用  
&nbsp;  
与参数无关的代码分离出template.
非类型模板参数可以用函数参数或类成员变量代替，有时候类型参数在底层有相同的二进制表达，比如所有的指针，此时可以用`void*`来代替强类型指针，减少代码膨胀    
&nbsp;  
运用成员函数模板接受所有兼容类型。  
模板具现化后的类之间是相互独立的，所以并不会发生隐式转换，可以用成员函数模板来进行编写泛化copy构造函数和泛化=操作符进行类型兼容(注意此时需要声明正常的copy构造函数和=操作符以免自动生成)  
&nbsp;  
需要类型转换时为模板定义非成员函数。  
在模板类中，有时有需要隐式转换支持的函数，问题在于template实参推导过程从不考虑隐式类型转换(也很好理解，要是考虑起来就会有非常多不必要的可能性)，将此函数定义为函数内部的friend函数(在类内实现，或者使用辅助函数减少inline开销)  
&nbsp;  
使用`traits classes`表现类型信息  
如果想在编译器获得类型相关信息，使用模板和模板特化完成。  
traits是一种技术，要求对自定义类型和内置类型的表现一样  
使用trait class应该整合函数重载和函数模板，令每个参数与traits信息相应和，然后建立一个控制函数根据traits信息调用响应函数。  
&nbsp;  
认识`template元编程`(TMP)  
TMP主要是个函数式语言，计算阶乘代码:
```c++
template <unsigned n>
struct Factorial
{
	enum { value = n * Factorial<n - 1>::value };
};

template<>
struct Factorial<0>
{
	enum { value = 1 };
};
```
把enum hack当int用，这是TMP基础技术。  
TMP将工作从运行期移到编译期，错误侦察更早，执行效率更高。  
### 定制new和delete
了解new-handler行为  
new-handler : 错误处理函数
set_new_handler : 指定错误处理函数
定制每个class专属的new-handlers只需要提供`set_new_handler`和`operator new`  
nothrow new用处较为局限，只能检测内存分配，如果之后的构造函数申请出现错误便无法察觉  
&nbsp;  
了解new和delete的合理替换时机  
定制new和delete的情况 : 
1. 检测可能的错误:例如超额分配内存来检测是否存在指针越界
2. 强化效能:定制的分配有时比通用的更强
3. 收集使用的统计数据:即分配的细节，例如分配算法之类的
...  
&nbsp;  
编写new和delete时需要固守常规  
operator new内应该包含一个无限循环来进行内存申请，如果无法满足则调用new-handler,申请字节为0时也要进行考虑(可以将0替换为1)  
operator delete应该处理null时直接返回，还要进行大小错误时的处理  
&nbsp;  
若有placement delete也要有placement new  
> placement new : 参数除了size还有其他的  (placement : 特定位置)

### 杂项
**不要忽视编译器警告**  
&nbsp;  
熟悉标准程序库(包括TR1)  
&nbsp;  
熟悉boost[boost.org]

\#############  
感谢陪伴!这确实是一本很精彩的书.我想未来我可能还会重温一下.