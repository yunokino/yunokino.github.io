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

};
```
>太妙了!利用了编译器和自动生成函数的特性将派生类自动生成函数的行为拒绝，利用'错误'进行禁止！

为多态基类声明virtual析构函数

别让异常逃离析构函数

 


