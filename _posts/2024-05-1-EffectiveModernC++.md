---
layout: post
title:  "Effective Modern C++"
---
## 一、类型推导
### 条款1: 模板类型推导
```c++
template<typename T>
void f(ParamType param);

f(expr);
```
考虑以上代码，在编译期编译器会推导**T**和**ParamType**的类型。  
1. ParamType是个指针或引用(非万能引用)  
若expr有引用类型将引用部分忽略，然后对expr和ParamType进行模式匹配来决定T的类型(指针同理)
2. ParamType是个万能引用  
若expr是左值，T和ParamType都被推导为左值，若expr是右值，应用情况1的规则
```c++
template<typename T>
void f(T&& param);

int x = 27;
f(x);//T=>int& , ParamType=>int&
f(27);//T=>int, ParamType=>int&&
```
3. ParamType既非指针也非引用  
此时即为按值传递，无论传入什么，param都只是一个副本，是一个全新的对象。若expr有引用类型将引用部分忽略，同样忽略const,volatile部分
```c++
template<typename T>
void f(T param);

f(expr);
```

函数或者数组类型的expr会退化为指针，除非被用来初始化引用(即形参是引用类型)

### 条款2: auto类型推导
auto与模板类型推导基本一致
```c++
template<typename T>
void f(ParamType param);

f(expr);
auto x = 27;
```
auto => T, x的类型=>ParamType  
只有一种情况例外:`initializer_list<T>`方法不能直接传递给模板
```c++
template<typename T>
void f1(T param);

template<typename T>
void f2(initializer_list<T> param);

f1({1,2,3});//编译错误
f2({1,2,3});//编译通过,T=>int
auto x {1,2,3};//编译通过,auto=>initializer_list<int>
```
在C++14中，返回值使用auto和在lambda表达式使用auto指定形参时都是用模板表达式推导，不可以直接用`{}初始表达式` 
### 条款3: decltype类型推导
decltype的主要用途是为了声明返回值类型依赖于形参的函数模板
```c++
template<typename Container, typename Index>
auto auth(Container c, Index i) -> decltype(c[i])
{
    ...
    return c[i];
}
```
使用auto做占位符，表明使用`返回值类型尾序语法`，其优点在于可以使用形参来指定返回值类型，因为auto可以自动推导可以删去后置返回，但是因为auto推导的内容会去掉&，所以返回的是一个右值，与预期不符(使用decltype后置可以)。解决办法是decltype(auto)
```c++
elem e;
const elem &ce = e;
auto e1 = ce;//e1's type=>elem
decltype(auto) e2 = ce;//e2's type=>const elem&
```
decltype返回的是名称的`声明类型`，但是只要一个左值表达式不仅是T，其就会返回T&，有时错误.example:
```c++
decltype(auto) f()
{
    int x=0;
    return (x);//decltype((x))=>int&,返回类型为int&,指向未定义行为
}
```
### 条款4: 查看类型推导结果的方法
1. IDE查看
2. 编译期诊断信息
3. 运行时输出,即使用typeid函数产生的typeinfo对象中的name函数，但是其处理对象的方式类似于向函数模板按值传递参数，引用,const,volatile将被忽略，可以使用boost库的typeindex替代
## 二、auto关键字
### 条款5: 优先使用auto而非显式型别声明
避免未初始化，复杂的变量声明，直接持有闭包(相较于function实例占据更少内存)
### 条款6: auto不符合要求时,显式声明
隐式代理类会导致auto得不到想要的结果，例如`vector<bool>`取[]，实际上是一个`vector<bool>::reference`对象向bool的隐式转换，直接使用auto会导致之后使用的时候对象已经析构产生未定义行为。  
可以使用static_cast转换类型强制得到对应类型
## 三、转向现代C++
### 条款7: 创建对象时区分()和{}
对于int之类的内置类型区分()和{}没必要，但用户定义的对象却需要，=一般是调用复制函数。  
{}禁止内建类型进行narrowing conversion(丢失信息的强转)。  
{}还可以避免解析语法:任何可以解析为声明的都要解析为声明，如`Widget w2()`本意调用`Widget`的无参构造初始化w2，却被解析为声明w2函数，而`Widget w2{}`正常。  
但是函数重载使用`{}`会优先匹配参数为`initializer_list`的函数，且优先级很高，即使元素类型不同也会进行转化(无相应转化可能时会正常)，如果是窄化则会报错。  
() or {} 在使用模板参数时会造成结果不同，例如创建`std::vector<int>`，(10,20)是10个20，而{10,20}是10和20
### 条款8: 优先使用nullptr,而非 0 or NULL
`nullptr`不具备整型类型，实际类型是`std::nullptr_t`，可以隐式转换到所有的裸指针型别。  
特别是关于模板推导时，nullptr会被推导为空指针而NULL和0为整型  
### 条款9: 优先使用别名声明，而非typedef
`using`可以使用模板化，而`typedef`不行  
带依赖型别前面要加`typename`标明是一个类型(有时候可能是个数据成员)，而别名模板可以免写`typename`
### 条款10: 优先使用限定作用域的枚举类型，而非不限类型的
使用**枚举类**限定枚举作用域(因为变量就不能再跟枚举中的变量取一样的名字)
```c++
enum class Color{black, white};
auto white = true; // 合法
```
而且枚举类中的变量不会隐式转换到整型(可以强转)，不过一切枚举类型在C++底层都是通过整数类型来实现  
枚举类另一个好处可以总是可以使用前置声明(默认int)，而不限作用域的因为需要确认底层整型类型所以一般不行(除非声明时指定类型)
```c++
enum Color : std::uint8_t;
```
不限类型的枚举的应用在取`tuple`时标明会比较有效，如
```c++
using Info = std::tuple<std::string, std::int>;
enum infoField { Name, Phone};
Info info_instance;
auto name = std::get<Name>(info_instance);
```
### 条款11: 优先使用删除函数，而非private未定义
任何函数都可以删除，包括非成员函数和模板具现。可以用来阻止特性类型的转换和模板偏特化
```c++
bool guess(int num);
bool guess(char c)=delete;
guess('a');// wrong
/***********************/
template<typename T>
void f(T a);

template<>
void f(int a)=delete;
```
private未定义的适用范围少而且有时只能在链接阶段报错
### 条款12: 为合适的函数添加override声明
C++11引入了引用饰词表示调用的(*this)指针是左值或右值时调用对应版本
```c++
class Widget {
public:
    void dosth() &; // 左值调用
    void dosth() &&;// 右值调用
}
```
添加override声明有利于查找错误
### 条款13: 优先使用const_iterator，而非iterator
优先选用非成员函数模板的cbegin和cend，返回对应的const_iterator，因为支持内建数组
### 条款14: 只要不会引起异常，优先加上noexcept
noexcept可以带有条件式，当条件式为真时才会noexcept
```c++
template<class T, size_t N>
void swap(T(&a)[N], T(&b)[N]) noexcept(noexcept(swap(*a, *b)));
```
### 条款15: 尽可能使用constexpr
constexpr对象具备const属性，且在编译阶段已知 
- constexpr函数的实参值在编译期已知，则结果同样在编译期计算得到
- 若实参有的在编译期未知，则与普通函数一致

constexpr的作用语境更广，所以在允许的情况下应该尽可能使用
### 条款16: 保证const成员函数的线程安全性
const可能修改mutable成员，所以要注意线程安全性，使用互斥量或者原子量(后者可能性能更好)
### 条款17: 理解特种成员函数的生成机制
特种成员函数 : C++会自动生成的成员函数  
移动操作只有类中未包含显式声明的复制、移动和析构时才会生成(此时移动会调用复制函数，但明显效率下降)  
复制构造函数只有类中不包含显式复制构造函数时会生成，复制赋值函数同理。如果有显式析构函数，同样不会生成复制操作  
成员函数模板不会阻止特种成员函数的生成
## 四、智能指针
### 条款18: 使用unique_ptr管理独有所有权的资源
unique_ptr大小与裸指针一致，可以自定义删除器，用lambda表达式不会占用额外字节，unique_ptr转化为shared_ptr也很方便

工厂模式是`unique_ptr`典型的应用场景  
`unique_ptr`可以使用`[]`管理数组，但并不推荐，因为std::array和std::vector已经足够(shared_ptr不能管理数组)
### 条款19: 使用shared_ptr管理共享所有权的资源
- `shared_ptr`的内部包含一个指涉资源的裸指针和一个指涉到引用计数的裸指针，是一般指针大小的两倍
- 引用计数的内存必须动态分配
- 引用计数的递增递减必须是原子操作  
- unique_ptr的析构器是指针类型的一部分而shared_ptr却不是(意味着shared_ptr即使使用不同的析构器的指针之间依然可以互相赋值)
- 所谓‘指涉到引用计数的裸指针’指向的其实是一个shared_ptr的控制块(位于堆上)，包含了众多数据
- 如果使用一个裸指针创造多个对应的shared_ptr就会有多个控制块，可能造成多次析构，最好就是直接传递new出来的对象
- 如果把this指针传入shared_ptr容器，那么会发生未定义行为
```c++
class Widget;
std::vector<std::shared_ptr<Widget>> vec;
Widget::process()
{
    vec.emplace_back(this); //!!!
}
```
如上所示，如果此Widget已经被一个shared_ptr所指涉，那么上述process将会导向未定义行为  
此时可以使用`enable_shared_from_this`为继承来的基类提供一个模板，使用其中的`shared_from_this()`接口使得this指针创建一个shared_ptr指针且不创建新的控制块只是引用计数加一
```c++
class Widget : public std::enable_shared_from_this<Widget> {
public:
    void process()
    {
        vec.emplace_back(shared_from_this());
    }
}
```
### 条款20: 使用weak_ptr管理可能悬置的指针
weak_ptr既不能提领，也不能检查是否为空，只是shared_ptr的一种扩充  
`expired()`接口可以检测是否失效，但是并发情况下有问题。`lock()`接口可以返回对应的shared_ptr，失效则为空  
`weak_ptr`可以解决循环引用问题，也已用来检测已失效的shared_ptr，对象大小与shared_ptr相同
### 条款21: 优先使用make_unique和make_shared，而非直接new
示例代码:
```c++
auto upw1(std::make_unique<Widget>());
std::unique_ptr<Widget>upw2(new Widget);
```
- 使用make函数，要写的代码更少
- make函数异常安全，传递给函数的实参在调用前完成求值，但是顺序不确定，如`process(std::shared_ptr<Widget>(new Widget), fun())`，`(new Widget)`创建后托管给共享指针之间可能会调用fun()，抛出异常后就会有内存泄漏，而make函数没有此问题
- 性能提升，new会引起两次内存分配，一次管理的对象和一个控制块，而make函数只会有一次
- 问题是一、make函数无法自定义析构器，二、如果类自定义了new和delete，那么无法正确使用make函数，三、使用make函数创建对象时只有当引用计数和弱引用计数同时为0时才析构内存，使用new的话可以将对象(引用计数为0)和控制块(弱引用计数和引用计数为0)的析构分开，对于对象内存较大时有意义
### 条款22: 使用Pimpl用法时，将特殊成员函数的定义放到实现文件中
使用`unique_ptr`实现Pimpl指针，需要在头文件中声明特种函数，但在实现文件实现，因为析构器是`unique_ptr`的一部分，需要知道类型，声明了析构也要声明其他的
## 五、右值引用、移动语义和完美转发
### 条款23: 理解std::move和std::forward
std::move不做移动，std::forward也不做转发，两者都只进行强制类型转换 
```c++
template <typename T>
typename remove_reference<T>::type&& 
move(T&& param)
{
    using ReturnType = typename remove_reference<T>::type&&;
    return static_cast<ReturnType>(param);
}
```
1. 如果对对象执行移动，其不能为const，因为const不能移动，所以最终调用的还是复制  
2. move不移动任何东西，也不保证其移动过的对象可移动
---
std::forward是一个有条件的类型转换，为了区分函数重载时的左值和右值
```c++
void process(const Widget& lval);
void process(Widget&& rval);

template<typename T>
void log(T&& param)
{
    process(std::forward<T>(param));
}
```
因为函数形参都为左值，所以不使用`forward`时都只会调用左值版本，而forward将信息编码在模板T形参中，将函数形参左值根据实参类型进行转换使得可以调用对应的重载版本 
### 条款24: 区分万能引用和右值引用
万能引用和右值引用都是`T&&`，但是万能引用可以应用于任何情况。万能引用的作用场景如下
```c++
template<typename T>
void f(T&& param); //万能引用

Widget&& var1 = Widget();
auto &&var2 = var1; // 万能引用
```
也就是说，涉及到**类型推导**时就是万能引用，其他情况下是右值引用，并且必须得`T&&`形式才可以
### 条款25: 对右值引用使用std::move，对万能引用使用std::forward
- 对右值引用的最后一次使用std::move，对万能引用的最后一次使用std::forward
- 如果局部对象可能用于返回值优化，不要用。因为RVO要求一、局部对象类型与函数返回值相同，二、返回的是局部对象本身，即使都不满足，返回对象也应当做右值处理

### 条款26: 避免按照万能引用类型重载
万能引用成为重载函数后几乎总可能会被调用，完美转发构造函数同样，因为会生成更加匹配的函数，当然传入的参数是const一般选用复制构造函数，因为模板实例化函数的优先级低于常规函数。
### 条款27: 使用万能引用类型重载的替代方案
1. 舍弃重载
2. 传递`const T&`类型的形参
3. 传值，调用move函数
4. 标签分派，true和false都是运行期值，而函数重载需要在编译器决定，所以`std::false_type/std::true_type`是对应的标签，`std::remove_reference_t<T>()`负责移除类型附加的一切引用附词
```c++
template<typename T>
void logAndAdd(T&& name)
{
	logAndAddImpl(
		std::forward<T>(name),
		std::is_integral<std::remove_reference_t<T>()>
	);
}

template<typename T>
void logAndAddImpl(T&& name, std::false_type)
{
	// ... 
}

template<typename T>
void logAndAddImpl(T&& name, std::true_type)
{
	// ... 
}
```
5. 对接受万能引用的模板加以限制
使用enable_if_t来禁用一些模板的生成,`std::decay_t<T>`可以去除移除了T的引用和const/volatile附词，`is_base_of`判断person是否是T的子类，综合意思就是禁止子类调用时和整型时的模板
```c++
class Person
{
public:
	template<
		typename T,
		typename = std::enable_if_t<
			!std::is_base_of<Person, std::decay_t<T>>::value
			&&
			!std::is_integral<std::remove_reference_t<T>>::value
			>
		>
		explicit Person(T&& n) :name(std::forward<T>(n))
	{
		// ...
	}
};
```
### 条款28: 理解引用折叠
引用折叠出现在四种情况下: 模板实例化，auto类型生成，创建和使用typedef和别名声明亦即decltype  
当出现引用的引用时，结果是单个引用，若原始的引用含有左值引用结果为左值引用，否则是右值引用  
模板推导的过程中如果传递的实参是个左值，那么T就是个左值引用，若是右值，则是非引用类型(即原始类型)
### 条款29: 假定移动操作不存在、成本高、未使用
对移动语义的使用保守一些，但对于类型一致的代码可以尽量用
### 条款30: 熟悉完美转发的失败情形
```c++
f(expr);
fwd(expr);//转发函数
```
当使用相同参数时上述两个函数调用的操作不一致，则为完美转发失败，情况如下： 
- 大括号初始化物。直接向未声明`std::initializer_list`的函数模板中传递大括号初始物，在此种**非推导语境**下，编译器禁止退到相应类型，但是可以使用auto绕路，先用大括号初始化auto局部变量，此时局部变量类型为`std::initializer_list`，但可以进行类型推导，可以进行转发
- 0和NULL推导为整型
- 仅有声明的整型static const成员变量，此时此变量无内存，仅当成常量处理。如果使用fwd进行转发会要求引用，此时自然失败
- 重载的函数名字和模板名字。因为不清楚转发到哪个重载版本或者模板函数实例，所以转发失败
- 位域。C++规定非const引用不得绑定到位域，自然无法使用转发
## 六、lambda表达式
### 条款31: 避免默认捕获模式
捕获模式有两种：按引用或按值，默认捕获模式是引用。  
按引用捕获可能会捕获已经析构的对象  
按值捕获同样可能会受空指针影响，特别是this指针  
lambda只能捕获非静态局部变量    
### 条款32: 使用初始化捕获(广义lambda捕获)将对象移入闭包
如下:
```c++
auto func = [pw = std::make_unique<Widget>()]{
    return pw->is_valid();
}
```
C++11中可以使用bind函数达到同样的效果
### 条款33: 对auto&&类型的形参用decltype，以std::forward之
泛型拉某大表达式=>可以在形参中使用auto。实现方法是闭包类的`operator()`使用模板实现 
```c++
auto f = [](auto&& param)
{
    return func(normalize(std::forward<decltype<param>>(param)));
}
```
### 条款34: 优先使用lambda表达式，而不是std::bind
lambda表达式比std::bind可读性更强，性能可能更好
## 七、并发API
### 条款35: 优先选用基于任务而非基于线程的程序设计
```c++
// 基于任务:
auto fut = std::async(dosth);
//基于线程
std::thread t(dosth);
```
std::thread的PI并未提供获取返回值的方法，也不能很好地处理异常(一般直接终止)，而std::async可以，同时其还可以自动管理线程耗尽、超订、负载均衡等情况  
除非应用于性能优化等情况，一般使用std::async
### 条款36: 若异步是必须的，则指定std::launch::async
`std::launch`有`std::launch::async`和`std::launch::defered`两种情况。前一种意味着其必须异步执行，后一种只有使用get或wait要求结果时才会运行，且是同步运行。  
默认策略是**或**运行，由系统决定，会出现一系列问题(比如对thread_local变量使用的不确定性)
### 条款37: 让std::thread类型所在对象在所有路径上均不可联结
如果线程析构时不可联结，那么会调用terminate函数使得程序终止，所以应该保持其不可联结
### 条款38: 对线程句柄析构函数保持关注
期值的析构函数一般只析构期值的成员变量

### 条款39: 考虑对一次性事件通信使用以void为模板类型实参的期值
条件变量可能冗余，使用`std::promise<void> p`可以回避此问题

### 条款40: 对并发使用std::atomic，对特种内存使用volatile
atomic对象是原子执行的  
volatile用于读写不可以被优化掉的内存
## 八、微调
### 条款41: 针对可复制的形参，在移动成本低且一定会被复制的情况下，考虑按值传递

如标题所言，当然基类类型不适用(因为切片问题，就是类的大小不一致，传递时会缺少一部分)

### 条款42: 考虑置入而非插入
就是使用emplace而不是push
