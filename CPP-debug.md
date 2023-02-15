---
title: CPP debug(GDB/LLDB)
date: 2023-02-10 11:47:26
tags:
    - CPP
---
# GDB
一些基础功能
```
Breakpoint, watchpoint, tracepoint, catch exception
Stepping and running
Registers, stack and backtrace, local variable, args
Symbol based information: structure, macro, variable, function and class
Print, dump, search and modify code and data
Log save, branch trace/full execution recording, python extension
````

- ~/.gdbinit, set auto-load safe-path ~/(To allow loading python extensions)/ set print pretty on/ set print array on

- kill -11 (pid) 得到 coredump, kill -11、kill -9 这些数字实际上代表的是信号的值：`#define SIGKILL     9/#define SIGSEGV     11`, 9 表示的是强制终止进程，SIGKILL信号不能被屏蔽，不能被忽略；
11 表示的是强制生成coredump文件，相当于是向未定义的内存去写入数据。“CTRL + C” 相当于是 发送 SIGINT(2) 。

- gdb --arg xxx, 然后 r(run) 开始运行, 使用start指令启动程序，完全等价于先在main()主函数起始位置设置一个断点，然后再使用run指令启动程序

- ctrl+c 停下来 debug

- info program 显示程序状态信息，是否在执行，进程是什么，暂停原因等

- b Aggregator.cpp:1845, 添加断点, 观察点（watchpoint）是一类特殊断点, 捕捉点（catchpoint）是另外一类特殊断点，当某种特定的事件发生时暂停程序执行，比如C++异常，加载动态库等

- tbread args         // 临时断点，只使用一次


- info breakpoints/ i b

- delete breakpoint

- next命令（可简写为n）用于在程序断住后，继续执行下一条语句

- step命令（可简写为s），它可以单步跟踪到函数内部

- continue命令（可简写为c）或者fg，它会继续执行程序，直到再次遇到断点处

- finish，当不小心单步进入了原本希望单步越过的函数时，使用fin返回

- bt 看堆栈
    - info args, 看当前栈帧的 args
    - info locals, 看当前的局部变量
    - info registers, 看寄存器

- up/down 上下堆栈, f frame_id 进入堆栈

- set (var) 修改变量的值

- set scheduler-locking off|on|step 是否让所有的线程在gdb调试时都执行
    - off ：（缺省）不锁定任何线程，也就是所有线程都执行
    - on ：只有当前被调试的线程能够执行
    - step ：阻止其他线程在当前线程【单步调试】的时候（即step时），抢占当前线程。只有当next、continue、util、finish的时候，其他线程才能重新运行。

- 指定线程可以使用inferior-num.thread-num的语法，两个参数分别为线程的ID，以及线程中的线程id。如果gdb中只有一个线程，那么gdb不会显示inferior-num。

- info inferiors 查看进程信息

- info threads 查看线程信息

- thread (n) 切换到某个线程上去, 注意参数 n 是gdb中的序号(1, 2, 3...)，而不是 LWP 的tid (16088等)

- watch ：为表达式expression设置一个观察点，一旦表达式值发生变化，马上停住程序/rwatch ：当表达式被读时，停住程序/awatch ：当表达式被读或写时，停住程序

- 禁/启用断点：disable(dis)/enable(dis)

- 处理信号(handle SIGUSR1 nostop)：
    - handle [actions]：收到signals时采取行动actions，signals可以是一个信号范围，actions可以是：
    - stop：收到该信号时，GDB会停住程序
    - nostop：收到信号时，GDB不会停住程序，但是会打印消息告诉你收到该信号
    - print：收到信号时，打印一条消息
    - noprint：收到信号时，GDB不会高告诉你收到信号
    - pass/noignore：收到信号时，GDB不做处理，让程序的信号处理程序接手
    - nopass/ignore：收到信号时，GDB不会让程序看到整个信号
    - info signals
    - info handle

- 自动打印：要是在每次程序停住的时候，能自动帮你打印变量的值，可以大大减少手工输入，display就可以做到。
    - display [/format] ：参数意义同print
    - undisplay ：删除自动显示，display_num类似断点的编号。
    - delete display ：display_num_list是空格分开的display_num列表
    - disable display ：和断点类似
    - enable display ：和断点类似

- 设置变量：set value=11：设置变量value的值为11

- 方便变量：`(gdb) set $i = 0 (gdb) p arr[$i++]``, 有时候想挨个打印数组的值，如果GDB能提供一个变量作为数组的下标，随着循环的进行变量值也随着变化，这样查看数组元素的值就非常方便了。

- shell command-string  //可以直接执行`command-string`所指定的命令而不需退出gdb。

- aliases -- Aliases of other commands                    // 别名

- ps -aux 查看到的Linux进程的几种状态：
R   :   running，正在运行 或 在运行队列中等待
S   :   sleeping，正在休眠（例如阻塞挂起）
T   :   停止或被追踪（停止：进程收到SIGSTOP、SIGSTP、SIGINT、SIGOUT信号后停止运行）
Z   :   僵尸进程（进程已终止，但进程描述符存在，直到父进程调用wait4()系统调用后释放）
D   :   不可中断（收到信号也不会唤醒或运行，进程必须等待中断发生）

- 线程的查看：
ps -aux  | grep a.out       //查看进程
ps -aL  | geep a.out        //查看线程
pstree -p (主线程ID)           //查看线程关系，树状图



