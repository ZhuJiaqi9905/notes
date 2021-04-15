# gdb

[TOC]

参考：

https://wizardforcel.gitbooks.io/100-gdb-tips/content/set-program-args.html

## 信息显示

启动时不显示信息：在`~/.bashrc`中，为gdb设置一个别名：`alias gdb="gdb -q"`

## 打开gdb

### 打开程序

```
gdb <program> 调试程序
gdb <program> core 用gdb同时调试一个运行程序和core文件，core是程序非法执行后core dump后产生的文件。
```

```
-symbols <file>
-s <file> 从指定文件中读取符号表
-se file 从指定文件中读取符号表，并把它用在可执行文件中
```

```
-core <file>
-c <file> 调试时core dump的core文件
```

```
-directory <directory>
-d <directory> 添加一个源文件的搜索路径。默认是环境变量中PATH所定义的路径。
```

可以在gdb启动时，通过选项指定被调试程序的参数，例如：

```
$ gdb -args ./a.out a b c
```

### 运行前设置

```
set args 设置命令行参数
show args 显示设置的命令行参数
```

```shell
path <dir> 设置程序的运行路径
show paths
```

```
set environment varname [=value] 设置环境变量 set env USER=zjq
show environment [varname]
```

```
pwd 显示当前所在目录
cd <dir>
```

```
info terminal 显示你程序用到的终端模式
run > outfile 重定向输出
tty命令可以指定输入输出的终端设备 tty /dev/ttyb
```

### 调试已运行的程序

```
方法1: gdb <program> PID 挂接到正在运行的程序
方法2: gdb <program>关联上源代码，并进行gdb。在gdb中用attach命令挂接进程的PID。并用detach取消挂接的进程。
```

### 执行shell命令

```
shell <command string> 例如shell clear
```

gdb还可以执行make指令

```
make <make-args>
等价于shell make <make-args>
```

### 清屏

`[ctrl] + L`或者`shell clear`

## 暂停程序

### 断点

```
break <function>
c++中用class::function或function(type,type)的格式
break filename:function
```

```
break 在下一条指令停住
break <linenum> 指定行号
break filename:linenum
b +offset  在当前行的后offset行加断点
b -offset
```

```
b *address 在程序的内存地址处停住
```

```
break ... if <condition>   ...是上述参数，condition是条件。例如 break 64 if i==100
```

```
查看断点, n是断点号
info breakpoints [n]
info break [n]
```



### 观察点



### 捕捉点



### 停止点



### 断点菜单

C++中存在函数重载。你可以通过指定参数类型的方式指定到对应的函数。否则GDB会给你列出一个断点菜单。你只需要输入菜单的编号即可。

```
(gdb) b String::after
[0] cancel
[1] all
[2] file:String.cc; line number:867
[3] file:String.cc; line number:860
[4] file:String.cc; line number:875
[5] file:String.cc; line number:853
[6] file:String.cc; line number:846
[7] file:String.cc; line number:735
> 2 4 6
Breakpoint 1 at 0xb26c: file String.cc, line 867.
Breakpoint 2 at 0xb344: file String.cc, line 875.
Breakpoint 3 at 0xafcc: file String.cc, line 846.
Multiple breakpoints were set.
Use the "delete" command to delete unwanted
breakpoints.
(gdb)
```



## 调试

```
continue [ignore-count]
c
fg
这三个命令是一样的
```

```
step <count> 会进入函数。前提是此函数被编译有debug信息。
s
set step-mode on 在进行但不跟踪时，程序不会因为没有debug信息而不停下。有利于查看机器码。
set step-mode off
```

```
next <count> 不会进入函数。
n
```

```
finish 运行程序，直到当前函数完成返回。
f 缩写好像没有用
```

```
until 厌倦在循环体单步调试时，可以运行程序直到退出循环体
u
```

```
stepi  单步跟踪一条机器指令。
si
nexti
ni
```



## 查看运行时数据

### 查看数据

```
print <expr>
p <expr>
p /<f> <expr>  <f>是输出格式。例如/x是16进制输出
p func 执行函数（方便地使用print调试）
```

### 输出格式

```
x 十六进制
d 十进制
u 十六进制无符号
o 八进制
t 二进制
a 十六进制
c 字符格式
f 浮点数格式

例如：p /f i
```

### 程序变量

可以查看：

1. 全局变量
2. 静态全局变量
3. 局部变量

如果全局变量和局部变量冲突，可以用`::`操作符。gdb能自动识别`::`是否为C++的操作符。

```
file::variable               例如：p 'f2.c'::x
function::variable
```

如果添加了编译优化，可能无法访问某些变量。

### 表达式

表达式语法应该是当前调试语言的语法。

`@`	一个与数组有关的操作符

`::`	指定一个在文件或是函数中的变量

`{<type>} <addr>`	表示一个指向内存地址`<addr>`的类型为`<type>`的对象。

### 数组

使用`@`操作符。`@`左边是第一个内存地址，右边是想要查看的内存长度。

```
如果有 int* array = (int*) malloc(len * sizeof(int));

(gdb) p *array@len
```

如果是静态数组，直接`print`数组名即可。

### 查看内存

### 查看寄存器

### 自动显示

设置自动显示的变量或表达式。

```
display <expr>
display/<fmt> <expr>
display/<fmt> <addr>

display/i $pc  $pc是GDB环境变量，表示指令的地址。/i表示格式为机器指令码。于是源代码和机器指令码对应。
```

```
info display
```

```
undisplay <dnums...>          
delete display <dnums...>
删除自动显示。dnums为标号。如果同时删除多个，用空分割。如果删除一个范围内的，用-分割(1-3)。
```

```
disable display <dnums...>
enable display <dnums...>
失效和恢复自动显示
```

### 设置显示选项