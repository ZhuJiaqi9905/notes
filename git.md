[TOC]



# git

## 常用操作

`git init`： 初始化一个仓库。

创建`.gitignore`文件。在其中编辑来添加要忽略的文件。

`git add .`：将所有非忽略的文件加入暂存区。

`git commit -m "message"`：提交代码到版本控制中。

`git status`: 查看仓库的状态。

配置用户名和邮箱：

- `git config user.name xxx`
- `git config user.email xxx`

## commit时记录详细信息

执行`commit -m`命令。然后编辑器会启动。记录详细信息格式如下：

- 第一行：简述更改内容
- 第二行：空行
- 第三行之后：记述更改的原因及详细内容

如果想终止提交。将提交信息留空并关闭编辑器。

## 分支

`git branch`: 查看分支。

`git branch feature-A` : 创建分支。

`git checkout feature-A`: 切换进`feature-A`分支。

`git checkout -b feature-A`: 等价于上面两条语句

`git branch -m old_name new_name`：把`old_name`分支重命名为`new_name`

`git branch -d dev` 删除分支，必须要和上游分支merge，,如果没有上游分支,必须要和`HEAD`完全merge。而``git branch -D dev`能强制删除分支。

## 远端仓库

### push

```
git push -u origin master:test
```

把本地master分支上的内容推送到远程origin的test分支上

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

查看本地分支和远端分支的对应关系

```
git branch -vv
```



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

## 远程分支

```
$ git remote rm upstream
$ git remote add upstream https://github.com/Foo/repos.git
$ git remote set-url upstream https://github.com/Foo/repos.git
```



### pull

```
git pull <远程主机名> <远程分支名>:<本地分支名>
```

Git也允许手动建立追踪关系。

```shell
$ git branch --set-upstream-to=origin/<branch> release
```

上面命令指定本地的`release`分支追踪到远程的`origin/<branch>`分支。

撤销与远程分支的联系

```
git branch --unset-upstream
```

git远端有很多分支时，git clone 得到主分支。

然后git branch -a显示所有分支（包括远端的）

git checkout -b zjq/misra origin/zjq/misra

新建本地分支并且和远端分支关联上



```
git remote -v
git fetch origin master 获取远端origin的master分支
git log -p master..origin/master 查看本地master与远端master的差异
git merge origin/master 合并远端orgin的master分支到当前分支
```

## git log

`git log`

`git log --oneline`

`git log -n 1`: 最近的1个commit

`git log --all`: 所有的git记录，包括不同分支的全部都显示

`git log --graph`:图形显示



## git commit

`git commit --amend`: 更改最新一次的commit message

`git rebase`: 变基

- `git rebase -i 版本号`：产生一个交互界面，在里面进行交互。

