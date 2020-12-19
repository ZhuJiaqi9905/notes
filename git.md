# git

## 常用操作

`git init`： 初始化一个仓库。

创建`.gitignore`文件。在其中编辑来添加要忽略的文件。

`git add .`：将所有非忽略的文件加入暂存区。

`git commit -m "message"`：提交代码到版本控制中。

`git status`: 查看仓库的状态。

## commit时记录详细信息

执行`commit -m`命令。然后编辑器会启动。记录详细信息格式如下：

- 第一行：简述更改内容
- 第二行：空行
- 第三行之后：记述更改的原因及详细内容

如果想终止提交。将提交信息留空并关闭编辑器。

## 分支

- `git branch`: 查看分支。

- `git branch feature-A` : 创建分支。

- `git checkout feature-A`: 切换进`feature-A`分支。

- `git checkout -b feature-A`: 等价于下面两条语句

```
git branch feature-A
git checkout feature-A
```



```
git push -u origin master:test
```

把本地master分支上的内容推送到远程的test分支上

```
对某个文件取消跟踪

git rm --cached readme1.txt    删除readme1.txt的跟踪，并保留在本地。

git rm --f readme1.txt    删除readme1.txt的跟踪，并且删除本地文件。

对文件夹及其下所有文件取消跟踪
git rm --cached -r cpp/out/
```

查看git追踪的文件

#### `git ls-tree -r master --name-only`

–name-only选项能使结果看起来简洁些

`git ls-files`

## .gitignore文件

有些文件是不应该让git进行版本控制的，例如编译生成的文件，可执行文件等。

一定要养成写`.gitignore`文件的习惯

官网给了很多模板https://github.com/github/gitignore

### 语法

- 空行或是以`#`开头的行是注释行。

- 可以在前面添加`/`来避免递归。

- 可以在后面添加正斜杠`/`来忽略一个文件夹，例如`build/`即忽略build文件夹。

- 可以使用`!`来否定忽略，即比如在前面用了`*.apk`，然后使用`!a.apk`，则这个a.apk不会被忽略。

- `*`用来匹配零个或多个字符，如`*.[oa]`忽略所有以".o"或".a"结尾，`*~`忽略所有以`~`结尾的文件（这种文件通常被许多编辑器标记为临时文件）；`[]`用来匹配括号内的任一字符，如`[abc]`，也可以在括号内加连接符，如`[0-9]`匹配0至9的数；`?`用来匹配单个字符。

例如：

```shell
# 忽略 .a 文件
*.a
# 但否定忽略 lib.a, 尽管已经在前面忽略了 .a 文件
!lib.a
# 仅在当前目录下忽略 TODO 文件， 但不包括子目录下的 subdir/TODO
/TODO
# 忽略 build/ 文件夹下的所有文件
build/
# 忽略 doc/notes.txt, 不包括 doc/server/arch.txt
doc/*.txt
# 忽略所有的 .pdf 文件 在 doc/ directory 下的
doc/**/*.pdf
```



## 如果忘记加.gitignore文件并补加

- 首先查看一下git跟踪了哪些文件
- 如果之前有文件是不想跟踪的，但是已经被跟踪了，要先对其取消跟踪。
- 然后再`git add .`和`git commit -m " "` 

例如：之前跟踪了`cpp/out`目录下的文件，现在想取消跟踪

```shell
$ git rm -r --cached cpp/out/
$ git add .
$ git commit -m "update .gitignore"
```

暴力一点的，可以先把本地所有跟踪的都取消跟踪。

```shell
$ git rm -r --cached .
$ git add .
$ git commit -m 'update .gitignore'
```

