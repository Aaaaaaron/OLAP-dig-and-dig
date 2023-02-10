---
title: Linux 初养成
date: 2019-01-05 10:32:31
tags:
  - Linux
  - Tool
---
# Shell-snipaste

1. `!$` : 代表上条命令的最后一个参数.
2. `ssh-copy-id -i ~/.ssh/id_rsa.pub root@host`: 快速与root@host建立免密连接
3. `lsof -i :7070 | awk 'NR==2 {print $2}' | xargs kill -9` : 杀死占用端口为7070的进程
4. `for i in {1..10}; do ll | wc -l; sleep 2; done` : 打印当前系统中句柄的打开数量
5. `ll | awk '{print $9}'` : 只取文件名
6. `find * | grep .DS_Store |xargs rm -rf` : 删除所有 .DS_Store
7. `ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime` : 切换时区
8. `pbcopy < ~/.ssh/id_rsa.pub` : 将公钥内容复制到剪切板
9. `du -hsx * | sort -rh | head -n` : 打印当前目录下占磁盘空间最多的n个文件
10. `/var/log` : 各个组件的 log, /var/log/messages 内核日志在这里.
11. CPU占用最多的前10个进程：
```
ps auxw|head -1;ps auxw|sort -rn -k3|head -10
top （然后按下M，注意大写）
```

12. 内存消耗最多的前10个进程
```
ps auxw|head -1;ps auxw|sort -rn -k4|head -10
top （然后按下P，注意大写）
```

13. 虚拟内存使用最多的前10个进程
`ps auxw|head -1;ps auxw|sort -rn -k5|head -10`

14. 配置 Linux 自启动 :
    a. 把命令加到 `/etc/rc.local`
    b. 将脚本放到目录 /etc/profile.d/ 下，系统启动后就会自动执行该目录下的所有shell脚本
    c. cp启动文件到 /etc/init.d/或者/etc/rc.d/init.d/（前者是后者的软连接）, `chkconfig --add cloudera-scm-agent`, `chkconfig cloudera-scm-agent on`, `chkconfig --list cloudera-scm-agent`

15. 关闭防火墙 : `service iptables status`, `service iptables stop`, `chkconfig iptables off`, `service iptables status`

16. 卸载 rpm 包 :
```
rpm -qa|grep -i java
rpm -e --nodeps xxx yyy zzz
```

17. du -sh, du -h --max-depth=1, df -h

18. `find / -type f -name "*.log" | xargs grep "ERROR"` : 统计所有的log文件中，包含Error字符的行

19. `ps -efL | grep [PID] | wc -l` : 查看某个进程创建的线程数

20. nohub

21. mount

22. `find .-name *iso >/tmp/res.txt &`, 加&放到后台执行, `bg` 可以查看后台运行的任务, `fg  %进程id` 放到前台执行
23. `find / -name zkCli.sh`: /opt/cloudera/parcels/CDH-5.11.2-1.cdh5.11.2.p0.4/lib/zookeeper/bin/zkCli.sh
也可以在cdh的节点上, 直接 z 然后 tab, 会发现有 zookeeper-client 这样一个命令.

![](Linux-初养成/9994c078-2f8a-40e8-9659-165afcde065d(1).png)

24. 找某个目录下是否有包含某各类的 jar 包: `grep -r "SparkSession" jars`
```shell
$ grep -r "SparkSession" jars
Binary file jars/spark-hive_2.11-2.2.1.jar matches
Binary file jars/spark-sql_2.11-2.2.1.jar matches
Binary file jars/hive-exec-1.2.1.spark2.jar matches
Binary file jars/spark-repl_2.11-2.2.1.jar matches
```

25. `hadoop classpath`, `/var/log`, `/usr/hdp/2.4.0.0-169/`

26. hive 默认 db 放的 HDFS 路径 `/user/hive/warehouse`, HIVE 查看表元数据: `desc formatted TEST_TIME`
27. `./spark-shell --master yarn --executor-cores 5 --executor-memory 5g --num-executors 3`
28. pip 切换 Python 版本

```python
cd /usr/local/bin
然后ls一下可能会发现好多pip文件，比如pip、pip2、pip2.7、pip3、pip3.5等等。使用一下命令：

file pip
输出：pip: Python script, ASCII text executable。发现pip其实是个Python脚本。于是我们用vim打开它：

#!/usr/bin/python3.6

# -*- coding: utf-8 -*-
import re
import sys

from pip._internal import main

if __name__ == '__main__':
    sys.argv[0] = re.sub(r'(-script\.pyw?|\.exe)?$', '', sys.argv[0])
    sys.exit(main())
看第一行，第一行指定了这个脚本对应的python版本，如果想把pip对应的python版本改为2.7的，只需修改python3.6为python2.7即可
```



***

1. `w !sudo tee %`: 就相当于从vim外部修改了当前编辑的文件(root修改只读文件)
2. grep 怎么样不出现 `grep --color=auto`, 加个[]: `ps -aux | grep "[c]lickhouse server" | awk '{print $2}' | xargs kill`
3. `socat TCP-LISTEN:2280,fork,reuseaddr TCP:ip:port` 端口转发, sudo sshuttle -v -r taojiatao@ip 0.0.0.0/0
4. 不包含 to_json 的文件列表, `ls |grep stack| xargs grep -L "to_json" | wc -l`
5. $ grep -Rn -i "current" *.sql | cut -d ":" -f 1 | sort | uniq | wc -l
6. Linux 两个文件求交集、并集、差集: 
   1. 交集: sort a.txt b.txt | uniq -d
   2. 并集: sort a.txt b.txt | uniq
   3. 差集: a.txt-b.txt: sort a.txt b.txt b.txt | uniq -u

7. git reset HEAD^
8. git 只保留 master: $ git branch | grep -v "(^*|master)" | xargs git branch -D
9. 在本机在远程执行命令: ssh tiger@xxx /opt/tiger/ss_bin/doas /opt/tiger/presto-bench/boot/run_local.sh restart
10. hdfs 打印二层目录: $ hadoop fs -ls /user/taojiatao/tpcds-text/tpcds-data-50g/ |awk -F" " '{print $8}' | xargs hadoop fs -ls | awk -F" " '{print $8}'
11. Scala test: mvn test -pl :spark-it -Dtest=moo -Dsuites='io.kyligence.kap.ut.TestScalaTest @query' -DfailIfNoTests=false
12. perf record -F 99 -p pid --call-graph dwarf  -- sleep 25
13. ps aux |grep '/data01/taojiatao/config/ch-config/' | grep -v grep | awk '{print $2}' | xargs kill -9
14. cat ~/hosts | while read host; do scp -o StrictHostKeyChecking=no ~/project/xxx tiger@$host:/opt/xxx/;done
15. 

## Java

1. jstat -gc [pid] : 查看gc情况
2. jmap -histo [pid] : 按照对象内存大小排序, 注意会导致full gc
3. jstack -l pid : 用于查看线程是否存在死锁

## shell 基础知识

```sh
$@  传递给脚本或函数的所有参数

if [ -f  file ]    如果文件存在
if [ -d ...   ]    如果目录存在
if [ -s file  ]    如果文件存在且非空 
if [ -r file  ]    如果文件存在且可读
if [ -w file  ]    如果文件存在且可写
if [ -x file  ]    如果文件存在且可执行   

if  [ -n $string  ]             如果string 非空(非0），返回0(true)  
if  [ -z $string  ]             如果string 为空
if  [ $sting ]                  如果string 非空，返回0 (和-n类似)    

```

|meta字符| meta字符作用|
|--------|-------------|
|= |设定变量|
|$ | 作变量或运算替换(请不要与`shell prompt`混淆)|命令
|>| 输出重定向(重定向stdout)|
|<|输入重定向(重定向stdin)|
|\||命令管道|
|&|重定向file descriptor或将命令至于后台(bg)运行|
|()|将其内部的命令置于nested subshell执行，或用于运算或变量替换|
|{}|将其内的命令置于non-named function中执行，或用在变量替换的界定范围|
|;|在前一个命令执行结束时，而忽略其返回值，继续执行下一个命令|
|&&|在前一个命令执行结束时，若返回值为true，继续执行下一个命令|
|\|\||在前一个命令执行结束时，若返回值为false，继续执行下一个命令|
|!|执行histroy列表中的命令|
|... | ...|

## snipaste

```sh
DIR="$( cd "$(dirname "$0")" ; pwd -P )"
```

$0 类似于python中的sys.argv[0]等。 $0指的是Shell本身的文件名。类似的有如果运行脚本的时候带参数，那么$1 就是第一个参数，依此类推。 
dirname 用于取指定路径所在的目录 ，如 dirname /home/ikidou 结果为 /home。 

$ 返回该命令的结果 

pwd -P 如果目录是链接时，格式：pwd -P 显示出实际路径，而非使用连接（link）路径。

############################################################################

`dir=$(cd -P -- "$(dirname -- "$0")" && pwd -P)`  会 cd 到脚本所在的目录

Note: A double dash (--) is used in commands to signify the end of command options, so files containing dashes or other special characters won't break the command.

Always use `-P` flag with `cd` command, alias cd='cd -P'

若将一个文件夹自己的快捷方式放到文件夹里,这样写脚本的时候就有可能会出现无限循环,当前路径名就会变得无限长,但是加上了-P命令后就可以避免无限循环的情况.

A
> ./myscript

B
> source myscript

Short answer: sourcing will run the commands in the current shell process. executing will run the commands in a new shell process. 


## variable
若从技术的细节来看，shell会依据IFS(Internal Field Seperator) 将command line所输入的文字拆解为"字段"(word/field)。 然后再针对特殊字符(meta)先作处理，最后重组整行command line。

变量替换(substitution) shell 之所以强大，其中的一个因素是它可以在命令行中对变量作 替换(substitution)处理。 在命令行中使用者可以使用$符号加上变量名称(除了用=定义变量名称之外)， 将变量值给替换出来，然后再重新组建命令行。

比方:

```
$ A=ls
$ B=la
$ C=/tmp
$ $A -$B $C
```

会得到:`ls -la /tmp`

echo命令只单纯将其argument送至"标准输出"(stdout, 通常是我们的屏幕)
```
$ echo $A -$B $C
>> ls -la /tmp

$ echo ${C}
>> /tmp
```

### export
严格来说，我们在当前shell中所定义的变量，均属于 "本地变量"(local variable), 只有经过export命令的 "输出"处理，才能成为"环境变量"(environment variable)：

## ()与{}差在哪？
要从一些命令执行的先后次序中得到结果， 如算术运算的2*(3+4)那样, 这时候，我们就可以引入"命令群组"(command group) 的概念：将许多命令集中处理。

*   `()` 将`command group`置于`sub-shell`(`子shell`)中去执行，也称 `nested sub-shell`。
*   `{}` 则是在同一个`shell`内完成，也称`non-named command group`。

在bash shell中, $()与``(反引号)都是用来做 命令替换(command substitution)的。

## Liunx 目录

- 通常从源码包编译安装的软件默认会放在/usr/local下
- `/etc/ld.so.conf` 这个文件记录了编译时使用的动态链接库的路径, 默认情况下，编译器只会使用**/lib**和**/usr/lib**这两个目录下的库文件 
- /etc, /bin, /sbin, /lib ：系统启动就需要用到的程序，这些目录不能挂载额外的分区，必须在根文件系统的分区上
- /usr/bin,/usr/sbin,/usr/lib：操作系统核心功能，可以单独分区
- /usr/local/bin,/usr/local/sbin,/usr/local/lib,/usr/local/etc,/usr/local/man：这个用于安装第三方程序，分别对应了二进制文件、库文件、配置文件、帮助文档的目录, 通常来说我们安装程序就安装在 /usr/local目录下
- 在传统的unix系统中，/usr通常只包含系统发行时自带的程序，而/usr/local则是本地系统管理员用来自由添加程序的目录
- *从用户权限的角度看*， /sbin目录下的命令通常只有管理员才可以运行，/bin下的命令管理员和一般的用户都可以使用。
- *从可运行时间角度看*， /sbin,/bin能够在挂载其他文件系统前就可以使用, /bin,/sbin目录是在系统启动后挂载到根文件系统中的，所以/sbin,/bin目录必须和根文件系统在同一分区。
- /opt这个目录是一些大型软件的安装目录，或者是一些服务程序的安装目录