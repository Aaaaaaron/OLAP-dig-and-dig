---
title: CPP 并发编程实战
date: 2022-10-06 21:25:25
tags: cpp
---

# 基础

## 启动线程

```cpp
void do_some_work();
std::thread my_thread(do_some_work);
```

当把函数对象传入到线程构造函数中时，需要避免“[最令人头痛的语法解析](http://en.wikipedia.org/wiki/Most_vexing_parse)”(*C++’s most vexing parse*, [中文简介](http://qiezhuifeng.diandian.com/post/2012-08-27/40038339477))。如果你传递了一个临时变量，而不是一个命名的变量；C++编译器会将其解析为函数声明，而不是类型对象的定义。可以使用大括号语法:`std::thread my_thread{background_task()}`

使用 lambda 表达式也可以避免这个问题:

```cpp
std::thread my_thread([]{
  do_something();
  do_something_else();
});
```

启动了线程，你需要明确是要等待线程结束(joined)/自主运行(detached)

## 线程管理

### 等待线程完成(join)

对于某个给定的线程，join()仅能调用一次；只要std::thread对象曾经调用过join()，线程就不再可汇合（joinable），成员函数joinable()将返回false

只要调用了join()，隶属于该线程的任何存储空间即会因此清除，std::thread对象遂不再关联到已结束的线程

#### 注意异常情况下的线程退出

如果线程启动以后有异常抛出，而join()尚未执行，则该join()调用会被略过(**需要注意**)

```cpp
class thread_guard
{
    std::thread& t;
public:
    explicit thread_guard(std::thread& t_):
        t(t_)
    {}
    ~thread_guard()
    {
        if(t.joinable())
        {
            t.join();
        }
    }
    thread_guard(thread_guard const&)=delete;
    thread_guard& operator=(thread_guard const&)=delete;
};  
```



### 后台运行线程(detach)

当t.joinable()返回true时，我们才能调用t.detach()

若不需要等待线程结束，我们可以将其分离, 分离操作会切断线程和std::thread对象间的关联

调用std::thread对象的成员函数detach()，会令线程在后台运行，遂无法与之直接通信。假若线程被分离，就无法等待它完结，也不可能获得与它关联的std::thread对象，因而无法汇合该线程(joinable为 false)

分离的线程确实仍在后台运行，其归属权和控制权都转移给C++运行时库（runtime library，又名运行库），由此保证，一旦线程退出，与之关联的资源都会被正确回收

分离出去的线程常常被称为守护线程（daemon thread）。这种线程往往长时间运行。几乎在应用程序的整个生存期内，它们都一直运行，以执行后台任务，如文件系统监控、从对象缓存中清除无用数据项、优化数据结构等。另有一种模式，就是由分离线程执行“启动后即可自主完成”（a fire-and-forget task）的任务；我们还能通过分离线程实现一套机制，用于确认线程完成运行。

## 传参

```cpp
void f(int i,std::string const& s);
std::thread t(f,3,"hello");
```

所有的传参, 都是按值传递

所以如果传的是指针, 一定要小心它指向的空间被提前 free, 造成悬垂指针

```cpp
void f(int i,std::string const& s);
void oops(int some_param)
{
    char buffer[1024];                  //    ⇽---  ①
    sprintf(buffer, "%i",some_param);
    std::thread t(f,3,buffer);          //    ⇽---  ②
	  std::thread t(f,3,std::string(buffer));     ⇽---  使用std::string避免悬空指针
    t.detach();
}
```



当我们需要传入引用的时候, 也会有问题, 对象会被复制, 如果直接写, update_data_for_widget() 函数调用会收到一个右值作为参数

```cpp
void update_data_for_widget(widget_id w,widget_data& data);  //    ⇽---  ①
```

可以看到 thread 的传参都是右值

```cpp
explicit thread(_Fp&& __f, _Args&&... __args);
```

需要这样传参:

```cpp
std::thread t(update_data_for_widget,w,std::ref(data));
```

## 所有权

std::thread类的实例能够移动却不能复制

对于任一特定的执行线程，任何时候都只有唯一的std:::thread对象与之关联，还准许程序员在其对象之间转移线程归属权

```cpp
void some_function();
void some_other_function();
std::thread t1(some_function);    //⇽---  ①
std::thread t2=std::move(t1);    //⇽---  ②
t1=std::thread(some_other_function);    //⇽---  ③
std::thread t3;    //⇽---  ④
t3=std::move(t2);    //⇽---  ⑤
t1=std::move(t3);    //⇽---  ⑥该赋值操作会终止整个程序
```

### 在运行时选择线程数量/识别线程

`std::thread::hardware_concurrency()`: 多核系统上，该值可能就是CPU的核芯数量

`识别线程std::this_thread::get_id()`: 识别线程
