---
title: Elixir 基础部分
date: 2018-07-19 20:44:00
tags: Elixir
---
# 0. 写在前面
## 编程时应该关注数据装换
用类和对象思考问题:类定义了行为,实例保存着状态.开发者构造类层次结构,为问题建模.OOP的时候,我们考虑的是状态,调用对象的方法和向某个对象传递其他对象.在这些调用中,对象更新自己或者其他对象的状态.类规范了每一个实例可以干什么,它是统治者,控制着实例的数据状态,目标是隐藏数据.

面向对象的一个缺点是 你想传入一个功能块(方法,函数),但是一定是要包在一个类里面,虽然有匿名函数,但是写法还是很丑,把函数当做一等公民(基本数据类型),直接传入函数的方式是最佳的.

真实世界里,并没有多少真正的层次结构,我们想要把事情搞定,而不是**维护状态**.我不要隐藏数据,我要转换数据.

## 借助管道来组合转换
UNIX的设计理念是将每个工具设计的小巧,功能单一,易于组合.每个工具都是获取输入,转换输入内容,并以下一个工具能使用的格式输出.这使得UNIX工具能一设计者预想不到的方式组合,而且这种模式还是高可靠的,每种工具之做好一件事,使之更容易测试.而且命令管道可以并行工作.

## 函数是数据转换器

函数越小巧,功能越单一,组合后的灵活性越大.(OO的一个设计理念是类之做好一件事,感觉都是共通的)

**数据转换的理念是函数式编程的核心**,当你不再肩负维护数据状态的责任,开始关注如何把事情做好的时候,你就会看到一个新世界

## 换一种方式思考
1. 程序的基础不是赋值,循环和if
2. 并发不一定需要锁,信号量,监视器等东西
3. 进行不一定要消耗大量的资源
4. 元编程不一定是语言的附属品
5. 编程即使是你的工作,也可以是充满乐趣的.记住**乐而为之**

# 1. 常规编程
## 模式匹配
在Elixir中,`=`不是赋值,而更像是一种断言(assertion).如果Elixir**可以找到一种方式**让等号的左边等于右边,则执行成功.Elixir把`=`称之为**匹配运算符**
如`list = [ 1, 2, 3 ]`,为了让匹配为真,Elixir将变量list绑定到列表[1,2,3].
如 `[a, b, c ] = list`,Elixir会试图让等号的左边等于右边.左边列表包含三个变量,右边列表包含3个值,所以将值设置到相应变量,等号两边才会相等.如:

```
iex> list = [1, 2, 3]
[1, 2, 3]
iex> [a, 2, b ] = list
[1, 2, 3]
iex> a
1
iex> b
3
```

这个很好的说明了是断言,而不是赋值,因为求出了a,b.这个叫做**模式匹配**

```
iex> x = 1
iex> 1 = x
```

`1 = x`也是成立的,这个在其他语言中是不行的,因为左右两边都是1,所以匹配.(elixir判断是否相等是用`===`和`==`)

模式匹配使得开发者能够简单地解构例如元组和列表的数据类型。在之后的章节中我们将看到这是Elixir中递归的基础，且其适用于其它类型，例如映射与二进制。

### 用_(下划线)忽略匹配值

`_`就像一个通配符,可以接受任何值.变量_的特别之处在于它永远不可以被读取

```
iex> [1, _, _] = [1, 2, 3]
[1, 2, 3]
iex> [1, _, _] = [1, "cat", "dog"]
[1, "cat", "dog"]
```

强制让变量的已有值参与匹配,而不是变成绑定新值(当你想要对变量值进行模式匹配，而不是重新赋值时)

```
iex> a = 1
1
iex> [^a, 2, 3 ] = [ 1, 2, 3 ] # use existing value of a
[1, 2, 3]
iex> a = 2
2
iex> [ ^a, 2 ] = [ 1, 2 ] #因为这里a用的是2,不能被赋值成2
```

## 从另一个角度看等号
Erlang的等号可以看成是代数里的等号,方程x = a + 1,不是将a + 1 赋值给x,而是断言x 和 a + 1 相等.一旦知道x,就能求出a,反之亦然.

模式匹配是Elixir的核心,将用它做条件判断,函数调用(function call),函数被调用(function invocation)

## 不可变性
**GOTO是邪恶的,因为我们会问:"我如何获得执行过程的入口点?",而可变性带给我们的问题则是:"我怎么样得到这个状态?"**

看下面的这个例子:

```
count = 99
doSometingWith( count )
print( count )
```

此时你还能确信你能输出99吗?不确定性是幸福感的最大杀手啊!更糟的是还有多线程,他们都修改这个count的话,画面太美不敢想象.

## 不可变数据才是已知的

**编程就是进行数据转换**,当更新[ 1, 2, 3 ]时,我们不是在原地修改它,而是将它转换成新数据.

## 不可变性对性能的影响

大家可能有个直觉复制是低效的,还会留下许多垃圾.事实上:**正因为某个数据永远都不会改变,所以可以简单的用来重用**

### 复制数据

```
iex> list1 = [ 3, 2, 1 ]
[3, 2, 1]
iex> list2 = [ 4 | list1 ]
[4, 3, 2, 1]
```

在可变的语言里,list2会新建一个列表,并把list1的值拷贝过来,但是在elixir中,它知道list1永远都不会改变,所以它简单的用4作为首项,把list1作为尾部创建一个新的列表.

### 垃圾回收
垃圾回收器影响性能,这个是共识了,但是在elixir中,他是以进程为基本单位的,而每个进程都有自己的堆,应用程序的数据由这些堆分摊,跟所以把所有数据放到一个堆里的情况比,每个单独的堆是很小的.因此,垃圾回收的速度会更快.并且,进程终止时,所有数据会被删除,没有必要垃圾回收.

## elixir基础

### 基本类型
```
iex> 1          # integer
iex> 0x1F       # integer
iex> 1.0        # float
iex> true       # boolean
iex> :atom      # atom / symbol,可以认为是常量,名字就是值,两个同名原子任何情况下都相等
iex> "elixir"   # string
iex> [1, 2, 3]  # list
iex> {1, 2, 3}  # tuple
```

**字符串**在Elixir内部被表示为二进制数值（binaries），也就是一连串的字节（bytes）：`iex> is_binary("hellö") >ture`; Elixir支持字符串插值（和ruby一样使用#{ ... }）：

```
iex> "hellö #{:world}"
"hellö world"
```

**列表与元组的区别**：

**列表**在内存中是以链表的形式存储的,元素值和指向下一个元素的指针为列表的一个单元(cons cell,就是有点类似 [ head | tail ] ).所以列表的前置拼接很快`[0] ++ list`,后置拼接就比较慢,但是要遍历就慢,比如获取长度,访问某个元素.

递归定义的列表是elixir编程的核心部分之一.

```
iex> list = [1|[2|[3|[]]]]
[1, 2, 3]
```

可以用`hd`和`tl`取出头尾,尝试从一个空列表中取出头或尾将会报错.

```
iex> list = [1,2,3]
iex> hd(list)
1
iex> tl(list)
[2, 3]
```

列表还有一个性能优势,要剔除列表的首部,只保留尾部,不需要拷贝列表,只要返回尾部的指针即可.

```
iex> [ 1, 2, 3 ] ++ [ 4, 5, 6 ] # concatenation
[1, 2, 3, 4, 5, 6]
iex> [1, 2, 3, 4] -- [2, 4] # difference
[1, 3]
iex> 1 in [1,2,3,4] # membership
true
iex> "wombat" in [1, 2, 3, 4]
false
```

**元组**
通常元组由两到四个元素组成,如果有更多元素,可以用散列表或者结构体
连续空间(数组),获取元组大小,或者使用索引访问元组元素的操作十分快速,添加修改元素开销大,因为这些操作会在内存中对元组的进行整体复制.

当需要计算某数据结构包含的元素个数时，Elixir遵循一个简单的规则： 如果操作在常数时间内完成（答案是提前算好的），这样的函数通常被命名为 *size。 而如果操作需要显式计数，那么该函数通常命名为 *length。

通常函数在不出错的情况下会返回一个元组

```
iex> {status, file} = File.open("mix.exs")
{:ok, #PID<0.39.0>}
```

所以有个惯用法是假定会成功的匹配

```
iex> { :ok, file } = File.open("Rakefile")
{:ok, #PID<0.39.0>} # 成功
** (MatchError) no match of right hand side value: {:error, :enoent} # 失败
```

**关键字列表**:k-v

散列表:map,`%{ key => value, key => value }%`

散列字典:HashDict,`<[ fg: "black", bg: "white", font: "Merriweather" ]>`

关键字列表:keyword, `[ fg: "black", bg: "white", font: "Merriweather" ]`,只有他允许一对多(k-v)

`[ name: "Dave", city: "Dallas", likes: "Programming" ]` elixir会转换成 `[ {:name, "Dave"}, {:city, "Dallas"}, {:likes, "Programming"} ]`

作为函数调用的最后一个参数的时候可以省略`[]` .例如`DB.save record, [ {:use_transaction, true}, {:logging, "HIGH"} ]`可以写成`DB.save record, use_transaction: true, logging: "HIGH"`

当我们有了一个元组（不一定仅有两个元素的元组）的列表，并且每个元组的第一个元素是个 原子， 那就称之为键值列表：

```
iex> list = [{:a, 1}, {:b, 2}]
[a: 1, b: 2]
iex> list == [a: 1, b: 2]
true
iex> list[:a]
1
```

实际上,这就是一个简单的列表而已,列表的操作都可以干.

关键字列表在任意期望列表值的上下文的最后一项出现,可以省略列表的括号.
```
iex> [1, fred: 1, dave: 2]
[1, {:fred, 1}, {:dave, 2}]
iex> {1, fred: 1, dave: 2}
{1, [fred: 1, dave: 2]}
```

特点:
1. 有序
2. key可以重复,重复key取值时，取回来的是第一个找到的（因为有序）

关键字列表在Elixir中一般就作为函数调用的可选项,或者传递一下参数,而想获得关联数组的话还是使用散列表.

**散列表**:k-v
和键值列表对比，散列表有两主要区别：
- 图允许任何类型值作为键
- 图的键没有顺序

如果你向散列表添加一个已有的键，将会覆盖之前的键-值对,一些例子:

```
iex> states = %{ "AL" => "Alabama", "WI" => "Wisconsin" }
%{"AL" => "Alabama", "WI" => "Wisconsin"}
# 用元组做key
iex> responses = %{ { :error, :enoent } => :fatal, { :error, :busy } => :retry }
%{{:error, :busy} => :retry, {:error, :enoent} => :fatal}
# 如果key是atom类型,可以像关键字列表那样的简略写法
iex> colors = %{ red: 0xff0000, green: 0x00ff00, blue: 0x0000ff }
%{blue: 255, green: 65280, red: 16711680}
# 访问散列表
iex> colors[:red]
16711680
iex> colors.green # 要是键是原子类型,还可以使用点符号.
6528
# key可以为不同的类型
iex> %{ "one" => 1, :two => 2, {1,1,1} => 3 }
%{:two => 2, {1, 1, 1} => 3, "one" => 1}
```

**还有系统类型**:PID,端口,引用,这些用于进程交互

## 匿名函数
匿名函数用fn创建,funName.(parm1, parm2, ...),点符号代表函数调用.

### 一个函数,多个函数体
单个函数定义中,定义不同的的实现,取决于传入的参数类型和内容,但不能根据参数数目进行选择,函数定义中的每个子句必须有相同数目的参数

简单来说,可以用模式匹配来选择要运行的子句(模式匹配很重要,有点难理解,但是一定要慢慢的理解),感觉就是if else的缩略版本?

```
iex> handle_open = fn Line 1
...> {:ok, file} -> "Read data: #{IO.read(file, :line)}" 2
...> {_, error} -> "Error: #{:file.format_error(error)}" 3
...> end 4

iex> handle_open.(File.open("code/intro/hello.exs")) # this file exists 6
"Read data: IO.puts \"Hello, World!\"\n" 7

iex> handle_open.(File.open("nonexistent")) # this one doesn't 8
"Error: no such file or directory" 
```

### 返回函数的函数

```
iex> fun1 = fn -> (fn -> "Hello" end) end
#Function<12.17052888 in :erl_eval.expr/5>

iex> other = fun1.()
#Function<12.17052888 in :erl_eval.expr/5>

iex> other.()
"Hello"
```

### 记住原始环境的函数(闭包)
```
iex> greeter = fn name -> (fn -> "Hello #{name}" end) end
#Function<12.17052888 in :erl_eval.expr/5>

iex> dave_greeter = greeter.("Dave")
#Function<12.17052888 in :erl_eval.expr/5>

iex> dave_greeter.()
"Hello Dave"
```

在返回`greeter.("Dave")`时,只是返回了一个函数,没有把name代换进去(所以这里不是把Dave放到name的时候),这里只返回了一个内部函数的定义(你就当做第一个fn就是返回了一个什么元素 只不过这个元素是个函数,没有深入到那个元素里面).当我们调用内部函数的时候,外部函数已经返回,参数的生命周期也已经终结.为啥内部函数还能取用到这个变量是因为这个作用域被绑定到外部函数上,当内部函数被定义的时候,这个绑定了name的作用域继承到了内部函数.这就是闭包-**作用于将其中的变量绑定封闭起来,并将它们打包到稍后能被保存并且使用的东西上**

### 参数化函数
```
iex> add_n = fn n -> (fn other -> n + other end) end
iex> add_two = add_n.(2)
iex> add_five = add_n.(5)

iex> add_two.(3)
5

iex> add_five.(7)
12
```

### 将函数作为参数传递
```
iex> list = [1, 3, 5, 7, 9]
[1, 3, 5, 7, 9]

iex> Enum.map list, fn elem -> elem * 2 end
[2, 6, 10, 14, 18]

iex> Enum.map list, fn elem -> elem * elem end
[1, 9, 25, 49, 81]

iex> Enum.map list, fn elem -> elem > 6 end
[false, false, false, true, true]
```

### &运算符
`add_one = &(&1 + &2)`相当于`add_one = fn (a,b) -> a + b end`
```
iex> Enum.map [1,2,3,4], &(&1 + 1)
[2, 3, 4, 5]
iex> Enum.map [1,2,3,4], &(&1 * &1)
[1, 4, 9, 16]
iex> Enum.map [1,2,3,4], &(&1 < 3)
[true, true, false, false]
```

## 模块与函数
```
defmodule Times do
  def double(n) do
    n * 2
  end
end
```

定义了Times模块,只有一个函数,这样调用`Times.double 4`

### 函数体是代码块
do...end不是真实的底层语法,真实的底层语法是`def double(n), do: n*2`,所以通常在单行代码中使用do:语法,多行使用do...end语法

### 函数调用与模式匹配
当调用一个命名函数的时候,elixir会尝试匹配第一个子句定义的参数列表,而过不匹配,尝试函数的下一个参数列表直到匹配完成(用来递归,你懂得,记住形参数量必须相等)

```
defmodule Factorial do
  def of(0), do: 1 # of是函数名
  def of(n), do: n * of(n - 1)
```

之后可以用尾递归改进这个实现,还有传入负数,会死循环,下节改

### 哨兵子句

由when紧接在函数定义之后的断言

```
defmodule Guard do
  def what_is(x) when is_number(x) do
    IO.puts "#{x} is a number"
  end
  
  def what_is(x) when is_list(x) do
    IO.puts "#{inspect(x)} is a list"
  end
  
  def what_is(x) when is_atom(x) do
    IO.puts "#{x} is an atom"
  end
end
```

改进factorial的死循环

```
defmodule Factorial do
  def of(0), do: 1
  def of(n) when n > 0 do
    n * of(n-1)
  end
en
```

注意,哨兵子句只支持elixir表达式的一个子集,详细可以看入门指南

### 默认参数

`param\\value`

例如

```
def func(p1, p2 \\ 2, p3 \\ 3, p4) do
  IO.inspect [p1, p2, p3, p4]
end
```

### 私有函数
`defp`仅能在声明它的模块被调用

### |> 管道运算符

|>获得左边表达式的结果,并将其作为第一个参数传给右边的函数调用.

```
filing = DB.find_customers
  |> Orders.for_customers
  |> sales_tax(2016)
  |> prepare_filing
```

`val |> f(a, b)` 等价于 `f(val, a, b)`

### 使用头部和尾部构造列表

```
def square([]), do: []
def square([ head | tail ]), do: [ head*head | square(tail) ]
```

可以看到这样的一个模式,真正工作的是第二个函数.并且会返回一个列表,把实际处理做在头部,并且处理的是head(因为tail可能还是一个列表,而head是一个离散的值),把递归调用写在尾部,以尾部作为参数来递归调用自身

### 创建映射函数(Map)

让我们对上面说的那个过程一般化,就是map函数.map函数是对一个collection的每个元素做运算.

```
def map([], _func), do: []
def map([ head | tail ], func), do: [ func.(head) | map(tail, func) ]
```

注意,这个head是每次处理的数据(用func),然后递归tail,tail下次就是一个完整的被做处理的.使用`MyList.map [1, 2, 3], fn (n) -> n*n end`

### 在递归过程中跟踪值

将状态传入函数的参数中

```
defmodule MyList do
  def sum([], total), do: total
  def sum([ head | tail ], total), do: sum(tail, head+total)
end
```

不过这样写必须传入一个0值`MyList.sum([1, 2, 3], 0)`,可以让模块只包含一个只接受一个列表的公开函数,调用私有函数来完成.

```
defmodule MyList do
  def sum(list), do: _sum(list, 0)

  defp _sum([], total), do: total
  defp _sum([ head | tail ], total), do: _sum(tail, head+total)
end
```

### Reduce模式

一个通用的函数,把一个collection规约成一个单个值(如求和,求最大,好像聚合操作)

```
defmodule MyList do
  def reduce( [], value, _ ) do
    value
  end
  
  def reduce( [head | tail], value, func ) do
    reduce( tail, func.( head, value ), func )
  end
end
```

使用`MyList.reduce( [1, 2, 3], 0, &( &1 + &2 ) )`

求最大值(**这个实现不错**)

```
defmodule MyList do
  def max( [ max ] ), do: max
  def max( [ max | [ head | tail ] ] ) when head > max, do: max( [head | tail] )
  def max( [ max | [ head | tail ] ] ) when head < max, do: max( [max | tail] )
end
```

### 更复杂的列表模式

`[ 1, 2, 3 | [ 4, 5, 6 ] ]`也可以,就是说`|`左边不一定只有一个元素

```
defmodule Swapper do
  def swap([]), do: []
  def swap([ a, b | tail ]), do: [ b, a | swap(tail) ]
  def swap([_]), do: raise "Can't swap a list with an odd number of elements"
end
```

` Swapper.swap [1,2,3,4,5,6]`可以调用成功,`Swapper.swap [1,2,3,4,5,6,7]`会抛出异常,因为第三个函数是处理一个参数时的情况,但是第二个参数每次要取两个值做运算,所以原始列表一定包含偶数个元素(不抛异常的话)

### 列表中的列表

```
defmodule WeatherHistory do
  def for_location_27([]), do: []
  def for_location_27([ [time, 27, temp, rain ] | tail]) do
     [ [time, 27, temp, rain] | for_location_27(tail) ]
  end
  def for_location_27([ _ | tail]), do: for_location_27(tail)
  
  def test_data do
  [
    [1366225622, 26, 15, 0.125],
    [1366225622, 27, 15, 0.45],
    [1366225622, 28, 21, 0.25],
    [1366229222, 26, 19, 0.081],
    [1366229222, 27, 17, 0.468],
    [1366229222, 28, 15, 0.60],
    [1366232822, 26, 22, 0.095],
    [1366232822, 27, 21, 0.05],
    [1366232822, 28, 24, 0.03],
    [1366236422, 26, 17, 0.025]
  ]
  end
end
```

使用`iex(9)> for_location_27(test_data)`,列表的头部必须是4个元素,第二个为27,其实就是`[|]`,|前面的规定格式,后面的是处理列表中的列表

有个改进的,那个27写死的不好,我们改成传入参数,再进一步的,把头部的写法也可以简略(因为过滤器只关心地区27,其他不关心),**重点**看`def for_location([ head = [_, target_loc, _, _ ] | tail], target_loc) do`的写法

```
defmodule WeatherHistory do
  def for_location([], _target_loc), do: []
  def for_location([ head = [_, target_loc, _, _ ] | tail], target_loc) do
    [ head | for_location(tail, target_loc) ]
  end
  def for_location([ _ | tail], target_loc), do: for_location(tail, target_loc)
end
```

## 复杂数据结构(Maps, Keyword Lists, Sets, and Structs)
- 散列表:map,`%{ key => value, key => value }%`,模式匹配用它
- 散列字典:HashDict,`<[ fg: "black", bg: "white", font: "Merriweather" ]>`,数量多用它
- 关键字列表:keyword, `[ fg: "black", bg: "white", font: "Merriweather" ]`,保证有序,只有他允许一对多(k-v)
这些api的文档需要花时间了解.

### 模式匹配和更新散列表

经常的用法:`person = %{ name: "Dave", height: 1.88 }`

1. 是否有一个entry的key是xxx?
```
iex> %{ name: a_name } = person
%{height: 1.88, name: "Dave"}
iex> a_name
"Dave"
```

2. 有key为xxx的entry吗?(有没有同时有这两个key)
```
iex> %{ name: _, height: _ } = person
%{height: 1.88, name: "Dave"}
```

3. 有keyw为xx的entry的值为xxx吗?
```
iex> %{ name: "Dave" } = person
%{height: 1.88, name: "Dave"}
```

**注意**,`%{ name: a_name } = person`这个destructured散列表的方法.这种方法很常用.


### 番外:类型是什么
keyword类型是elixir的模块,却被实现成了元组列表.显然它还是一个列表,其次elixir添加了字典的行为,从某种意义上来说,这是类似鸭子类型.keyword模块没有底层原生的数据类型,它只是假定所处理的数据都是按特定方式组织的列表(schema?依赖抽象,高层指定规范底层遵守?)

## 处理集合-Enum与Stream
Enum类似批处理,Stream是流处理.Enum是贪婪的,Stream是延迟处理的
从技术上讲,可被遍历的类型都被为实现了Enumerable.Enum用于迭代,过滤,组合,分割等.  
断言操作:

```
iex> Enum.all?(list, &(&1 < 4))
false
iex> Enum.any?(list, &(&1 < 4))
true
iex> Enum.member?(list, 4)
true
iex> Enum.empty?(list)
false
```

```
[ 1, 2, 3, 4, 5 ]
|> Enum.map(&(&1*&1))
|> Enum.with_index
|> Enum.map(fn {value, index} -> value - index end)
|> IO.inspect #=> [1,3,7,13,21]
```

# Actor综述
更加面向对象! 你所有的对象,最后都想成长为一个actor.

# 队列式信箱
异步地发送消息是用actor模型编程的重要特性之一。消息并不是直接发送到一个actor，而是发送到一个**信箱(mailbox)**

信箱可以存放许多消息(Buffered Channels?)

这样的设计解耦了actor之间的关系——actor都以自己的步调运行，且发送消息时不会被阻塞。

虽然所有actor可以同时运行，但它们都按照信箱接收消息的顺序来依次处理消息，且仅在当前消息处理完成后才会处理下一个消息，因此我们只需要关心发送消息时的并发问题即可

通常actor会进行无限循环，通过receive等待接收消息，并进行消息处理。为了彻底关闭一个actor，需要满足两个条件。第一个是需要告诉actor在完成消息处理后就关闭；第二个是需要知道actor何时完成关闭。

actor是异步发送消息的——发送者并不会被阻塞。

# 管理进程

任何时候都可以用Process.link()在两个进程之间建立连接,这样可以检测到某一个进程的终止.连接是双向的。建立了从pid1到pid2的连接的同时，也就建立了从pid2到pid1的连接——所以如果其中一个进程终止，那么两个进程就都终止了.进程正常终止(:normal)是不会让连接的另一个进程终止的.
```
defmodule LinkTest do
     def loop do
         receive do
             {:exit_because, reason} -> exit(reason)
             {:link_to, pid} -> Process.link(pid)
             {:EXIT, pid, reason} -> IO.puts("#{inspect(pid)} exited because #{reason}")
         end
         loop()
     end
end
```

## 系统进程

通过设置进程的:trap_exit标识，可以让一个进程捕获另一个进程的终止消息。用专业术语来说，这是将进程转化为系统进程：
```
def loop_system do
  Process.flag(:trap_exit, true)
  loop
end
```

实现一个进程管理者CacheSupervisor（也就是一个系统进程），它管理着若干个工作进程，当工作进程崩溃时进行干预。

CacheSupervisor负责创建Cache实例,如果缓存崩溃,会自动重启(虽然会丢失之前的数据),但至少得到了一个崩溃后可以继续使用的缓存

after是为receive增加超时机制,比如的活可能会有死锁
1. 进程1向缓存发送:put消息；

2. 进程2向缓存发送:get消息；

3. 缓存在处理进程1的消息时崩溃了；

4. 管理者将缓存重启，但进程2的消息丢失了；

5. 进程2在receive处陷入死锁，一直在等待消息的回复，但这个回复永远不会发送。
```
defmodule Cache do
  def start_link do
    pid = spawn_link(__MODULE__, :loop, [HashDict.new, 0])
    Process.register(pid, :cache)
    pid
  end

  def put(url, page) do
    send(:cache, {:put, url, page})
  end

  def get(url) do
    ref = make_ref()
    send(:cache, {:get, self(), ref, url})
    receive do
      {:ok, ^ref, page} -> page
      after 1000 -> nil
    end
  end

  def size do
    ref = make_ref()
    send(:cache, {:size, self(), ref})
    receive do
      {:ok, ^ref, s} -> s
      after 1000 -> nil
    end
  end

  def terminate do
    send(:cache, {:terminate})
  end

  def loop(pages, size) do
    receive do
      {:put, url, page} ->
        new_pages = Dict.put(pages, url, page)
        new_size = size + byte_size(page)
        loop(new_pages, new_size)

      {:get, sender, ref, url} ->
        send(sender, {:ok, ref, pages[url]})
        loop(pages, size)

      {:size, sender, ref} ->
        send(sender, {:ok, ref, size})
        loop(pages, size)

      {:terminate} -> # Terminate request - don't recurse
    end
  end
end

defmodule CacheSupervisor do
    def start do
        spawn(__MODULE__, :loop_system, [])
    end
    def loop do
        pid = Cache.start_link
         receive do
             {:EXIT, ^pid, :normal} -> 
                 IO.puts("Cache exited normally")
                 :ok
             {:EXIT, ^pid, reason} -> 
                 IO.puts("#{inspect(pid)} exited because #{reason} - restarting it")
            loop()
        end
     end
     def loop_system do
         Process.flag(:trap_exit, true)
         loop()
     end
end
```

# 错误处理内核模式(error-kernel)
> 软件设计有两种方式：一种方式是，使软件过于简单，明显地没有缺陷；另一种方式是，使软件过于复杂，没有明显的缺陷。

Elixir有两个规则：
- 如果没有异常发生，消息一定能被送达并被处理；
- 如果某个环节出现异常，异常一定会通知到使用者（假设使用者已经连接到或正在管理发生异常的进程）
第二条规则是Elixir提供容错性的基石。

一个软件系统如果应用了错误处理内核模式，那么该系统正确运行的前提是其错误处理内核必须正确运行。成熟的程序通常使用尽可能小而简单的错误处理内核——小而简单到明显地没有缺陷。

对于一个使用actor模型的程序，其错误处理内核是顶层的管理者，管理着子进程——对子进程进行启动、停止、重启等操作。

程序的每个模块都有自己的错误处理内核——模块正确运行的前提是其错误处理内核必须正确运行。子模块也会有自己的错误处理内核，以此类推。这就构成了错误处理内核的层级树，较危险的操作都会被下放给底层的actor执行(一棵树,高层的都是错误处理内核,是管理者,风险最低;子树是工作进程风险高)

## 任其崩溃
使用actor模型的程序并不进行防御式编程，而是遵循“任其崩溃”的哲学，**让actor的管理者来处理这些问题**。这样做有几个好处，比如：

1. 代码会变得更加简洁且容易理解，可以清晰区分出“一帆风顺”的代码和容错代码；
2. 多个actor之间是相互独立的，并不共享状态，因此一个actor的崩溃不太会殃及到其他actor。尤其重要的是一个actor的崩溃不会影响到其管理者，这样管理者才能正确处理此次崩溃；
3. 管理者也可以选择不处理崩溃，而是记录崩溃的原因，这样我们就会得到崩溃通知并进行后续处理。

## 小总结
Elixir通过创建管理者并使用进程的连接来进行容错：
1. 连接是双向的——如果进程a连接到进程b，那么进程b也连接到进程a；
2. 连接可以传递错误——如果两个进程已经连接，其中一个进程异常终止，那么另一个进程也会异常终止；
3. 如果进程被转化成系统进程，当其连接的进程异常终止时，系统进程不会终止，而是会收到:EXIT消息。

# OTP
这段代码中Cache声明自己实现了一个行为（GenServer.Behaviour）和两个函数（handle_cast()和handle_call()）。这里所说的“行为”非常类似于Java中的接口——其定义了一个函数集。模块使用use来声明自己实现了行为.

`handle_cast()`可以处理消息但并不回复消息。其接受两个参数：收到的消息、actor的当前状态。返回值是一个二元组`{:noreply, new_state}`。本例中实现了一个`handle_cast()`来处理:put消息。

`handle_call()`可以处理消息且回复消息。其接受三个参数：收到的消息、发送者标识、actor的当前状态。返回值是一个三元组`{:reply, reply_value, new_state}`。本例中实现了两个`handle_call()`，一个负责处理:get消息，另一个负责处理:size消息。类似于Clojure，Elixir用下划线`（_）`开头的变量名来表示该变量不被使用——比如`_from`。

```
defmodule Cache do
  use GenServer.Behaviour
  #####
  # External API

  def start_link do
    :gen_server.start_link({:local, :cache}, __MODULE__, {HashDict.new, 0}, [])
  end

  def put(url, page) do
    :gen_server.cast(:cache, {:put, url, page})
  end

  def get(url) do
    :gen_server.call(:cache, {:get, url})
  end

  def size do
    :gen_server.call(:cache, {:size})
  end

  #####
  # GenServer implementation

  def handle_cast({:put, url, page}, {pages, size}) do
    new_pages = Dict.put(pages, url, page)
    new_size = size + byte_size(page)
    {:noreply, {new_pages, new_size}}
  end
  def handle_call({:get, url}, _from, {pages, size}) do
    {:reply, pages[url], {pages, size}}
  end

  def handle_call({:size}, _from, {pages, size}) do
    {:reply, size, {pages, size}}
  end
end

defmodule CacheSupervisor do
  use Supervisor.Behaviour

  def start_link do
    :supervisor.start_link(__MODULE__, []) 
  end

  def init(_args) do
    workers = [worker(Cache, [])]
    supervise(workers, strategy: :one_for_one)
  end
end
```

## 重启策略
OTP管理者行为支持多种不同的重启策略，最常用的是one-for-all和one-for-one。

如果一个工作进程崩溃，使用one-for-all策略的管理者将重启所有工作进程（包括那些没有崩溃的工作进程）。使用one-for-one策略的管理者仅重启已经崩溃的工作进程。

用OTP实现的服务器和管理者有着更多的功能，其中包括以下几点。

1. 更好的重启逻辑： 之前我们自己实现的简单管理者使用非常草率的重启策略——如果工作线程崩溃，就将其重启。如果工作线程在启动时很快就崩溃，那么管理者会一直重启工作线程。而OTP提供的管理者可以设定最大重启频率，如果重启超过这个频率，管理者将会异常终止。
2. 调试与日志：通过调整OTP服务器的参数，可以开启调试和日志功能，这对开发很重要。
3. 代码热升级：OTP服务器不需要停止整个系统就可以进行升级。
4. 还有许多：发布管理、故障切换、自动扩容，等等。

## 复习
> 很久以前，我在描述“面向对象编程”时使用了“对象”这个概念。很抱歉这个概念让许多人误入歧途，他们将学习的重心放在了“对象”这个次要的方面。
真正主要的方面是“消息”……日文中有一个词ma，表示“间隔”，与其最为相近的英文或许是“ interstitial”。创建一个规模宏大且可生长的系统的关键在于其模块之间应该如何交流，而不在于其内部的属性和行为应该如何表现。

actor模型精心设计了消息传输和封装的机制，强调了面向对象的精髓，可以说actor模型非常“面向对象”。actor模型的重点在于参与交流的实体，而CSP模型的重点在于用于交流的通道。

# 运用多线程
**所有你的对象最后都想成长成actor**

elixir的一个特点就是将代码打包成可独立和并发运行的小块,它使用actor并发模型,actor是一个无依赖的进程,他不于其他进程共享任何东西.你可以spawn新进程,send发送消息,用receive接收消息,仅此而已.elixir里创建进程就和java中创建对象一样自然.

```
defmodule SpawnBasic do
  def greet do
    IO.puts "Hello"
  end
en
```

在独立的进程中运行,`:greet`是函数名,`[]`是参数列表;返回一个pid(整个是世界中唯一),先返回hello还是先返回PID是不确定的,要用消息同步进程的活动
```
iex> spawn(SpawnBasic, :greet, [])
Hello
#PID<0.42.0>
```

### 在进程间发送消息

`send`接受一个PID和右边发送的消息(也叫term),一般都是原子和元组;等待消息用`receive`,用法像case,消息体作为参数,可以指定模式.

下面的会输出Hello, world,`{ sender, msg }`是receive接受的参数,sender发过去的时候要带着.当发送第二个消息,不过greet函数仅能处理单条消息,处理完receive(Spaen1)就退出了.所以下面模式匹配的receive救护挂起,用after做一个超时机制

```
defmodule Spawn1 do
  def greet do
    receive do
      { sender, msg } ->
        send sender, { :ok, "Hello, #{msg} "}
    end
  end
end

# 客户端代码
pid = spawn( Spawn1, :greet, [] )
send pid, { self(), "world" } # send self(), "world",{ self, "world" }对应上面receive的{ sender, msg }

# 一个模式匹配,提取出message
receive do
  { :ok, message } ->
    IO.puts message
end

# 发送第二个消息,不过greet函数仅能处理单条消息,处理完receive(Spaen1)就退出了.所以下面模式匹配的receive救护挂起,用after做一个超时机制
send pid, { self(), "Aron" }

receive do
  { :ok, message } ->
    IO.puts message
  after 500 ->
    IO.puts "The greeter has gone away"
end
```

让greet函数处理多条消息.函数体就这么写,一个递归

```
def greet do
  receive do
    {sender, msg} ->
      send sender, { :ok, "Hello, #{msg}" }
      greet
  end
end
```

### 循环递归和栈

**尾递归优化:** 函数的最后一件事是调用自己,就没有必要调用,只需要简单的跳到函数开始的地方,如果函数调用有参数,就把原始参数替换掉.,但是递归调用必须是最后一个执行的,下面的方式就不行.因为他还要将之后的结果*n.注释下面的是可以进行尾递归优化的(把乘法移到了递归里面),要增加一个累加器.

```
def factorial(0), do: 1
def factorial(n), do: n * factorial(n-1)
#############################
def factorial(n), do: _fact(n, 1)
defp _fact(0, acc), do: acc
defp _fact(n, acc), do: _fact(n-1, acc*n)
```

### 进程开销


```
defmodule Chain do
  def counter( next_pid ) do
    receive do
     n ->
       send next_pid, n+1
     end
  end

  def create_processes( n ) do
    last = Enum.reduce 1..n, self,
                  fn ( _, send_to ) ->
                    spawn(Chain, :counter, [ send_to ])
                  end

    send last, 0

    recevie do
      final_answer when is_integer( final_answer ) ->
        "Result is #{ inspect( final_answer ) }"
    end
  end

  def run( n ) do
    IO.puts inspect :timer.tc(Chain, :create_processes, [n])
  end
end
```