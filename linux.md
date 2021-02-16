# linux

https://explainshell.com/

### 小贴士

用`clear`进行清屏

命令过长时，可以用`\[Enter]`来转义Enter实现换行

`[ctrl]+u/[ctrl]+k`：从光标处向前/后删除命令串

`[ctrl]+a/[ctrl]+e`：让光标移动带整个命令串的最前面/最后面

## shell

### bash shell

#### bash shell的功能：

- 历史命令：历史命令存放在`~/.bash_history`中，其中存储的是前一次登录之前执行过的命令。这次登录时执行过的命令存在内存中
- 命令与文件补全：[Tab]键
- 设置命令别名：

```shell
$ alias lm='ls -al'
```

命令行中使用`alias`来查询目前的命令别名。

- 任务管理、前台、后台控制
- 程序化脚本
- 通配符

#### 查询命令是否为bash shell的内置命令

查询命令name是否为内置命令

```shell
$ type [-tpa] name
```

### shell变量

变量可以定义在`~/.bashrc`中

`echo ${变量}`：读取变量内容

设置和修改变量的内容：`=`

- 等号两边不能有空格
- 变量内容如果有空格，可使用双引号或单引号
  - 双引号内的特殊字符如`$`，保有原本的特性：如`var="is $LANG"`相当于`var="is C.UTF-8"`
  - 单引号内的特殊字符如`$`，不保有原本的特性：如`var='is $LANG'`相当于`var='is $LANG'`
- 可用转义字符`\`
- 如果一串命令的执行需要借由其他额外的命令所提供的信息，可以用`` `命令` ``或者`$(命令)`：例如`version=$(echo "hello")`或者`` version=`echo "hello"` ``,得到`version="hello"`

- 若该变量为扩增变了内容时，可用`"$变量"`或`${变量}`累加内容：如`PATH="$PATH":/home/bin`或``PATH=${PATH}:/home/bin``

- 如果该变量需要在其他子程序中使用，则需要以export来使变量变成环境变量
- 通常大写字符为系统默认变量，小写为用户自定义的变量
- 用`unset 变量`来取消变量

```sh
#进入当前的内核模块目录
$ cd /lib/modules/$(uname -r)/kernel
```

#### 环境变量

环境变量能被子进程引用，其他自定义变量不会存在于子进程中。

`env`命令或`export`命令查看环境变量

`set`命令查看所有变量：环境变量和自定义变量

重要的系统内定需要的变量：

- PS1: 提示字符的设置
- $: 本shell的PID, `echo $$`

- ?: 关于上个命令的返回值, `echo $?`。一般来说上个命令如果成功执行，其返回值为0
- OSTYPE, HOSTTYPE, MACHTYPE: 主机硬件与内核的等级

影响显示结果的语系变量`locale`

#### 变量键盘读取

- `read`命令读取键盘输入的变量

```shell
$ read [-pt] variable
```

- `declare`或`typeset`声明变量的类型

```shell
$ declare [-aixr] variable

$ declare -i sum=100+50
```

变量默认为字符串。bash环境中数值运算仅能达到整数形态。

- 数组变量类型

```shell
$ var[index]=content
$ echo ${var[index]}
```

- `ulimit`限制用户的某些系统资源

```shell
$ ulimit [-SHacdfltu] [配额] #单位是KB
```

