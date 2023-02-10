---
title: C++ 右值/移动/完美转发
date: 2022-10-05 23:44:11
tags: cpp
---

# C++ 右值/移动/完美转发

总结: `std::move`并不移动任何东西; 完美转发也并不完美; 移动操作并不永远比复制操作更廉价, 而且它也并不总是被调用，即使在当移动操作可用的时候; 构造“`type&&`”也并非总是代表一个右值引用。

要牢记形参永远是**左值**，即使它的类型是一个右值引用: `void f(Widget&& w);`

## 引用

理解左右值之前, 最重要是理解引用, 可以把引用看做是通过一个常量指针来实现的, 但是你没法获得一个引用的地址, 编译器会自动转成引用指向的地址(所有对引用的操作都会转移到被引用的值上)，它只能绑定到初始化它的对象上

一旦引用已经定义, 它就不能再指向其他的对象, 这就是为什么引用要被初始化的原因, **指针可以为空，引用不能为空**

左值引用就是对左值进行引用的类型, 右值引用就是对右值进行引用的类型, 他们都是引用, 都是别名, 并不拥有所绑定对象的堆存, 所以都必须立即初始化

这句话是精髓: **把值变成引用, 不管是左值还是右值, 都避免了拷贝. 左值引用/右值引用避免拷贝的核心在"引用" , 而不是在左右值上**

```cpp
type &name = exp; // 左值引用
type &&name = exp; // 右值引用
```

```cpp
int &c = 10; // error，10无法取地址，无法进行引用
const int &d = 10; // ok，因为是常引用，引用常量数字，这个常量数字会存储在内存中，可以取地址(存疑)
```

## 左右值

<img src="./C++ 右值移动完美转发/18b692072537d4ce179d3857a8a0133c.png" alt="img" style="zoom:50%;" />

<img src="./C++ 右值移动完美转发/036cc6865a9623a48918b504e408945a-20221004235155403.png" alt="img" style="zoom:50%;" />

左值与右值的根本区别在于是否允许取地址(&运算符)获得对应的内存地址, 变量可以取地址, 所以是左值, 但是常量和临时对象等不可以取地址, 所以是右值(字符串常量除外). 也可以说保存在CPU寄存器中的值为右值，而保存在内存中的值为左值

在函数调用时, 左值可以绑定到左值引用的参数如 T&; 一个常量只能绑定到常左值引用, 如 const T&

左值 `lvalue` 是有标识符/可以取地址的表达式, 最常见的情况有:

- 变量/函数或数据成员的名字
- 返回左值引用的表达式，如 `++x、x = 1、cout << ' '`, 对于操作符``++、=、<< `等会返回左值引用
- 字符串字面量如 "hello world" (字符串字面量之所以是左值, 是因为它需要占用主存, 是可以取地址的; 而整数, 字符等可以直接放在寄存器, 不能取地址)

纯右值 prvalue 是没有标识符, 不可以取地址的表达式, 一般也称之为“临时对象”, 最常见的情况有：

- 返回非引用类型的表达式，如 `x++、x + 1、make_shared(42)`
- 除字符串字面量之外的字面量，如 42、true

`smart_ptr(smart_ptr<U>&& other)`: other 是一个**左值变量**, 其类型为**右值引用**. 因为他**是左值**(因为可以取地址), 所以用他做参数匹配调用的函数的是左值的签名, 这点切记

c++ 11 之前, 右值只可以绑定到常左值引用(无法取地址, 也不会修改)

c++ 11 之后, 右值可以绑定到右值引用(&&)

另外提一点: 在 C++ 里所有的原生类型、枚举、结构、联合、类都代表值类型, 只有引用（&）和指针（*）才是引用类型; 在 Java 里, 数字等原生类型是值类型, 类则属于引用类型; 在 Python 里一切类型都是引用类型, 这里想强调的是: **C++ 里的对象缺省都是值语义**

```cpp
// b_/c_都在栈上, Java 这俩都会是指针
class A {
  B b_;
  C c_;
};
```

**形参总是左值**

### 通用引用

通用引用有两个条件:

1. 必须是T&&的形式, 由于auto等价于T, 所以auto && 符合这个要求
2. T类型要可以推导, 也就是说它必须是个模板且一定要推导类型, 而auto是模板的一种变型
3. 模板方法的模板参数都可以从所在模板类推出来的就不是万能引用(由于在模板内部并不保证一定会发生类型推导)

一个通用引用的初始值决定了它是代表了右值引用还是左值引用

```cpp
template<typename T>
void f(T&& param);              //param是一个通用引用

Widget w;
f(w);                           //传递给函数f一个左值；param的类型
                                //将会是Widget&，也即左值引用

f(std::move(w));                //传递给f一个右值；param的类型会是
                                //Widget&&，即右值引用
```

下面代码有两种类型的通用引用: 一种是auto, 另一种是通过模板定义的T&&. 实际上auto就是模板中的T, 它们是等价的

```cpp
template<typename T>
void f(T&& param){
    std::cout << "the value is "<< param << std::endl;
}

int main(int argc, char *argv[]){

    int a = 123;
    auto && b = 5;   //通用引用，可以接收右值

    int && c = a;    //错误，右值引用，不能接收左值

    auto && d = a;   //通用引用，可以接收左值

    const auto && e = a; //错误，加了const就不再是通用引用了

    func(a);         //通用引用，可以接收左值
    func(10);        //通用引用，可以接收右值
}

void f(Widget&& param);             //右值引用
Widget&& var1 = Widget();           //右值引用
auto&& var2 = var1;                 //不是右值引用

template<typename T>
void f(std::vector<T>&& param);     //右值引用, 通用引用必须恰好为“T&&”, 当函数f被调用的时候，类型T会被推导,但是param的类型声明并不是T&&，而是一个std::vector<T>&&。这排除了param是一个通用引用的可能

template<typename T>
void f(T&& param);                  //不是右值引用

template <typename T>
void f(const T&& param);        //param是一个右值引用, 即使一个简单的const修饰符的出现，也足以使一个引用失去成为通用引用的资格

// 这个 T&& 是个右值引用, 不是通用引用
// 类实例化下来就确定的类型不是通用引用
// 一定要方法被调用的时候有类型推导的才是
template<class T, class Allocator = allocator<T>>   //来自C++标准
class vector
{
public:
    void push_back(T&& x);
    …
}

// Args是独立于vector的类型参数T的，所以Args会在每次emplace_back被调用的时候被推导
template<class T, class Allocator = allocator<T>>   //依旧来自C++标准
class vector {
public:
    template <class... Args>
    void emplace_back(Args&&... args);
    …
};
```

如果函数重载同时接受 右值引用/常引用 参数，编译器 **优先重载** 右值引用参数

```cpp
void f(const Data& data);  // 1, data is c-ref
void f(Data&& data);       // 2, data is r-ref

f(Data{});  // 2, prefer 2 over 1 for rvalue
```

**避免使用 常右值** 否则无法使用移动

此外，**类的成员函数** 还可以通过 [**引用限定符** *(reference qualifier)*](https://en.cppreference.com/w/cpp/language/member_functions#const-.2C_volatile-.2C_and_ref-qualified_member_functions)，针对当前对象本身的左右值状态（以及 const-volatile）重载：

```cpp
class Foo {
 public:
  Data data() && { return std::move(data_); }  // rvalue, move-out
  Data data() const& { return data_; }         // otherwise, copy
};

auto ret1 = foo.data();    // foo   is lvalue, copy
auto ret2 = Foo{}.data();  // Foo{} is rvalue, move
```



## 移动语义(std::move) & 完美转发

在运行时，它们不做任何事情。它们不产生任何可执行代码，一字节也没有

`std::move`和`std::forward`仅仅是执行转换（cast）的函数（事实上是函数模板）。`std::move`无条件的将它的实参转换为右值，而`std::forward`只在特定情况满足时下进行转换

**切记: 不要 move 一个常量(踩过坑)**, move常量作为参数调用的还是拷贝构造

复习下上面的精髓: **把值变成引用, 不管是左值还是右值, 都避免了拷贝. 左值引用/右值引用避免拷贝的核心在"引用" , 而不是在左右值上**

右值&移动语义最大的作用就是把以前的右值大对象, 变成了可以取引用的值, 这样传引用比传值更快, 就是这么简单, 上面说到C++ 里的对象缺省都是值语义, 好处是用到局部性, 坏处是大对象在栈上复制很占性能, **所以移动语义使得在 C++ 里返回大对象（如容器）的函数和运算符成为现实**, 而 Java 对象默认在堆上, 就是指针语义, 不存在这个问题, 不用 move

`std::move()`: 就是一个 static_cast, 把一个左值引用强制转换成一个右值引用，而并不改变其内容

```c++
shard_ptr<shape> ptr1{new A()};
shard_ptr<shape> ptr2 = std::move(ptr1);
```

`new A()` 就是一个纯右值(prvalue)；但对于指针，我们通常使用值传递，并不关心它是左值还是右值。

`std::move(ptr1);` 看作是一个有名字的右值。为了跟无名的纯右值 `prvalue` 相区别，C++ 里目前就把这种表达式叫做 `xvalue`

`prvalue/xvalue` 主要区别在生命周期, 如果一个 prvalue 被绑定到一个引用上, 它的生命周期则会延长到跟这个引用变量一样长; xvalue (move)无效, 所以切记, move 了一个对象之后, 就不要用他了(虽然可能暂时不会出问题, 但是是个 ub)

### 源码解析

```cpp
template<typename T>                            //在std命名空间
typename remove_reference<T>::type&& move(T&& param)
{
    using ReturnType =                          //别名声明，见条款9
        typename remove_reference<T>::type&&;

    return static_cast<ReturnType>(param);
}
```

`std::move`接受一个对象的引用（准确的说，一个通用引用（universal reference）, 返回一个指向同对象的引用。

该函数返回类型的`&&`部分表明`std::move`函数返回的是一个右值引用

但是如果类型`T`恰好是一个左值引用，那么`T&&`将会成为一个左值引用。`std::remove_reference`应用到了类型`T`上，因此确保了`&&`被正确的应用到了一个不是引用的类型上

`std::move`总是**无条件**的将它的实参为右值不同，`std::forward`只有在满足一定条件的情况下才执行转换

```cpp
void process(const Widget& lvalArg);        //处理左值
void process(Widget&& rvalArg);             //处理右值

template<typename T>                        //用以转发param到process的模板
void logAndProcess(T&& param)
{
    auto now = std::chrono::system_clock::now();
    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param));
}

Widget w;
logAndProcess(w);               //用左值调用
logAndProcess(std::move(w));    //用右值调用
```

开头说的, 函数里的 param 不管怎么样, 都是一个左值, 每次在函数`logAndProcess`内部对函数`process`的调用，都会因此调用函数`process`的左值重载版本。为防如此，我们需要一种机制：当且仅当传递给函数`logAndProcess`的用以初始化`param`的实参是一个右值时，`param`会被转换为一个右值, 这就是 forward.

#### C++ 14 move 实现

得益于函数返回值类型推导

```cpp
template<typename T>
decltype(auto) move(T&& param)          //C++14，仍然在std命名空间
{
    using ReturnType = remove_referece_t<T>&&;
    return static_cast<ReturnType>(param);
}
```

#### 不要 move 一个 const 值

move const 值不会报错, 也能运行, 但是行为可能让你无法理解, e.g.

```cpp
const string val = "xxx";
string val2(std::move(val)) ;
```

上面的代码调用的其实是拷贝构造函数

当编译器决定哪一个`std::string`的构造函数被调用时，考虑它的作用，将会有两种可能性：

```cpp
class string {                  
public:                         
    …
    string(const string& rhs);  //拷贝构造函数
    string(string&& rhs);       //移动构造函数
};
```

`std::move(val)`的结果是一个`const std::string`的右值。这个右值不能被传递给`std::string`的移动构造函数，因为移动构造函数只接受一个指向 `non-const`的`std::string`的右值引用

但是该右值却可以被传递给`std::string`的拷贝构造函数，因为lvalue-reference-to-`const`允许被绑定到一个`const`右值上。因此，`std::string`在成员初始化的过程中调用了\**拷贝**构造函数，即使`text`已经被转换成了右值。这样是为了确保维持`const`属性的正确性

从一个对象中移动出某个值通常代表着修改该对象，所以语言不允许`const`对象被传递给可以修改他们的函数

总结:

1 . 不要在你希望能移动对象的时候，声明他们为`const`。对`const`对象的移动请求会悄无声息的被转化为拷贝操作

2. `std::move`不仅不移动任何东西，而且它也不保证它执行转换的对象可以被移动。关于`std::move`，你能确保的唯一一件事就是将它应用到一个对象上，你能够得到一个右值

### 移动构造函数&拷贝构造函数

拷贝构造函数中，对于指针, 我们一定要采用深层复制; 而移动构造函数中, 对于指针我们采用浅层复制, 并把原对象的指针置为 NULL(避免两个指针共同指向一片内存空间), **否则第一个对象析构了, 第二个对象指向它的部分就会是一个非法访问**, 这就是移动构造和拷贝构造最大的不同: 移动构造函数应当从另一个对象获取资源，清空其资源，并将其置为一个可析构的状态

```cpp
A(A && a)
{
    ptr_ = a.ptr_;
    a.ptr_ = nullptr;
}
```

支持移动构造的对象, 应该有 swap 成员函数，支持和另外一个对象快速交换成员, swap在C++11的实现： `template<typename T> void swap(T& a,T&b) { T temp(std::move(a)); a = std::move(b); b = std::move(temp); } `移动方式的本质就是移交临时对象对资源的控制权，通常就是指针的替换

```cpp
void swap(smart_ptr& rhs) noexcept
{
  using std::swap;
  swap(ptr_, rhs.ptr_);
  swap(shared_count_,
       rhs.shared_count_);
}
```



#### 例子 1:

会调用 A 的一次构造方法, 一次拷贝构造方法

```cpp
int main(int argc, char *argv[]){
    std::vector<A> vec;
    vec.push_back(A());
}
```

会调用 A 的一次构造方法, 一次移动构造方法

```cpp
int main(int argc, char *argv[]){
    std::vector<A> vec;
    vec.push_back(std::move(A()));
}
```

#### 例子 2:

```cpp
Demo get_demo(){
    return Demo();
}
int main(){
    Demo a = get_demo();
    return 0;
}
```

执行流程(编译器未优化), 事实上优化过后的代码只有一次 construct

1. construct!                <-- 执行 Demo()
2. copy construct!       <-- 执行 return Demo()
3. class destruct!         <-- 销毁 Demo() 产生的匿名对象
4. copy construct!       <-- 执行 a = get_demo()
5. class destruct!         <-- 销毁 get_demo() 返回的临时对象
6. class destruct!         <-- 销毁 a

### 完美转发

左值引用一定是左值引用，右值引用就不一定是右值引用

std::forward被称为**完美转发**，它的作用是保持原来的`值`属性不变(左右值)

```cpp
template<typename T>
void print(T & t){
    std::cout << "lvalue" << std::endl;
}

template<typename T>
void print(T && t){
    std::cout << "rvalue" << std::endl;
}

template<typename T>
void testForward(T && v){
    print(v);
    print(std::forward<T>(v));
    print(std::move(v));
}

int main(int argc, char * argv[])
{
    testForward(1);
    std::cout << "======================" << std::endl;
    int x = 1;
    testFoward(x);
}

```

**执行结果**

```
lvalue
rvalue
rvalue
=========================
lvalue
lvalue
rvalue
```

从上面第一组的结果我们可以看到，传入的1虽然是右值，但经过函数传参之后它变成了左值（在内存中分配了空间）；而第二行由于使用了std::forward 函数，所以不会改变它的右值属性，因此会调用参数为右值引用的print模板函数；第三行，因为std::move会将传入的参数强制转成右值，所以结果一定是右值。

再来看看第二组结果。因为x变量是左值，所以第一行一定是左值；第二行使用forward处理，它依然会让其保持左值，所以第二也是左值；最后一行使用move函数，因此一定是右值

#### 原理

forward实现了两个模板函数，一个接收左值，另一个接收右值。在上面有代码中：

```cpp
template <typename T>
T&& forward(typename std::remove_reference<T>::type& param)
{
    return static_cast<T&&>(param);
}

template <typename T>
T&& forward(typename std::remove_reference<T>::type&& param)
{
    return static_cast<T&&>(param);
}
```

`typename std::remove_reference<T>::type`: 获得去掉引用后的参数类型

#### 例子 2

```cpp

void foo(const shape&)
{
  puts("foo(const shape&)");
}

void foo(shape&&)
{
  puts("foo(shape&&)");
}

void bar(const shape& s)
{
  puts("bar(const shape&)");
  foo(s);
}

void bar(shape&& s)
{
  puts("bar(shape&&)");
  foo(s);
}

int main()
{
  bar(circle());
}
```

输出为

```
bar(shape&&)
foo(const shape&)
```

如果我们要让 bar 调用右值引用的那个 foo 的重载，我们必须写成`foo(std::move(s));`

可如果两个 bar 的重载除了调用 foo 的方式不一样，其他都差不多的话，我们为什么要提供两个不同的 bar , 事实上，很多标准库里的函数，连目标的参数类型都不知道，但我们仍然需要能够保持参数的值类别：左值的仍然是左值，右值的仍然是右值

可以把两个 bar 函数简化成:

```cpp
// T 是模板参数时，T&& 的作用主要是保持值类别进行转发 aka万能引用(T&&作为模板参数时才是“万能引用”)
template <typename T>
void bar(T&& s)
{
  foo(std::forward<T>(s));
}

int main()
{
  circle temp;
  bar(temp);
  bar(circle());
}
```

输出是:

```
foo(const shape&)
foo(shape&&)
```





### 返回值优化(NRVO)

```cpp
class Obj {
public:
  Obj()
  {
    cout << "Obj()" << endl;
  }
  Obj(const Obj&)
  {
    cout << "Obj(const Obj&)"
       << endl;
  }
  Obj(Obj&&)
  {
    cout << "Obj(Obj&&)" << endl;
  }
};

Obj simple()
{
  Obj obj;
  // 简单返回对象；一般有 NRVO
  return obj;
}

Obj simple_with_move()
{
  Obj obj;
  // move 会禁止 NRVO
  return std::move(obj);
}

Obj complicated(int n)
{
  Obj obj1;
  Obj obj2;
  // 有分支，一般无 NRVO
  if (n % 2 == 0) {
    return obj1;
  } else {
    return obj2;
  }
}

int main()
{
  cout << "*** 1 ***" << endl;
  auto obj1 = simple();
  cout << "*** 2 ***" << endl;
  auto obj2 = simple_with_move();
  cout << "*** 3 ***" << endl;
  auto obj3 = complicated(42);
}
```

返回

```
*** 1 ***
Obj()
*** 2 ***
Obj()
Obj(Obj&&)
*** 3 ***
Obj()
Obj()
Obj(Obj&&)
```

## 常见误解总结

#### 误解: 返回前移动局部变量

`std::move()` 移动局部变量`，会导致后续代码不能使用该变量

#### 误解: 被移动的值不能再使用

被移动的对象进入一个 **合法但未指定状态** *(valid but unspecified state)*，调用该对象的方法（包括析构函数）不会出现异常，甚至在重新赋值后可以继续使用：

另外，基本类型（例如 `int/double`）的移动语义 和拷贝相同：

#### F.48: Don’t return std::move(local)

有了copy elision, 现在基本上不用在 return 上用 std::move

##### F.18: For “will-move-from” parameters, pass by X&& and std::move() the parameter

误解：不移动右值引用参数, 因为不论 **左值引用** 还是 **右值引用** 的变量（或参数）在初始化后，**都是左值**

- **命名的右值引用** *(named rvalue reference)* **变量** 是 **左值**，但变量类型 却是 **右值引用**
- 在作用域内，**左值变量** 可以通过 **变量名** *(variable name)* **被取地址、被赋值**

```cpp
std::unique_ptr<int> bar(std::unique_ptr<int>&& val) {
  //...
  return val;    // not compile
                 // -> return std::move/forward(val);
}
```

所以，返回右值引用变量时，需要使用 `std::move()/std::forward() `显式的 § 5.4 移动转发 或 § 5.3 完美转发，将变量 “还原” 为右值（右值引用类型）

#### 正确写移动构造函数

例如，标准库容器 `std::vector` 在扩容时，会通过 [`std::vector::reserve()`](https://en.cppreference.com/w/cpp/container/vector/reserve#Exceptions) 重新分配空间，并转移已有元素。如果扩容失败，`std::vector` 满足 [**强异常保证** *(strong exception guarantee)*](https://en.cppreference.com/w/cpp/language/exceptions#Exception_safety)，可以回滚到失败前的状态。

为此，`std::vector` 使用 [`std::move_if_noexcept()`](https://en.cppreference.com/w/cpp/utility/move_if_noexcept) 进行元素的转移操作：

- 优先 使用 `noexcept` 移动构造函数（高效；不抛出异常）
- 其次 使用 拷贝构造函数（低效；如果异常，可以回滚）
- 再次 使用 非 `noexcept` 移动构造函数（高效；如果异常，**无法回滚**）
- 最后 如果 不可拷贝、不可移动，**编译失败**

如果 没有定义移动构造函数 或 自定义的移动构造函数没有 `noexcept`，会导致 `std::vector` 扩容时执行无用的拷贝，**不易发现**。

## 引用:

https://time.geekbang.org/column/article/169268

https://blog.avdancedu.com/a39d51f9/

https://blog.avdancedu.com/360c1c76/

https://bot-man-jl.github.io/articles/?post=2018/Cpp-Rvalue-Reference

Effective modern c++