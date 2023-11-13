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
>auto_ptr在C++11后便因为出现问题后难以定位被废除，所以此处仅供了解
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


### 设计与声明