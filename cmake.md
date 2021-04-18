# CMake

[TOC]



## 基础

### 单个源文件编译

CMake语言不区分大小写，但是参数区分大小写。

#### Minimum CMake version

```cmak
cmake_minimum_required(VERSION 3.5)
```

#### 声明项目名称和语言

```cmak
project(hello_cmake LANGUAGES CXX)
```

#### create an executable

```cmake
add_executable(hello-world hello-world.cpp)
```

#### 源外构建

```shell
$ mkdir -p build
$ cd build
$ cmake ..
$ cmake --build .
```

上述内容相当于

```shell
$ cmake -H. -Bbuild
$ cmake --build .
```

该命令使用了`-H`和`-B`为CLI选项。`-H`表示当前目录中搜索根`CMakeLists.txt`文件。`-Bbuild`告诉CMake在一个名为`build`的目录中生成所有的文件。

#### 更多信息

CMake生成的目标比构建可执行文件的目标要多。可以使用`cmake --build . --target <target-name>`语法，实现如下功能：

* **all**(或Visual Studio generator中的ALL_BUILD)是默认目标，将在项目中构建所有目标。
* **clean**，删除所有生成的文件。
* **rebuild_cache**，将调用CMake为源文件生成依赖(如果有的话)。
* **edit_cache**，这个目标允许直接编辑缓存。

对于更复杂的项目，通过测试阶段和安装规则，CMake将生成额外的目标：

* **test**(或Visual Studio generator中的**RUN_TESTS**)将在CTest的帮助下运行测试套件
* **install**，将执行项目安装规则
* **package**，此目标将调用CPack为项目生成可分发的包

### 切换生成器

CMake能被不同平台和工具支持，输入`cmake -help`可以看到支持cmake的平台列表

例如，如果想使用Ninga generator

```shell
$ cmake -G Ninja ..
$ cmake --build .
```

相当于：

```shell
$ cmake -H. -Bbuild -GNinja
```

### 构建和链接静态库和动态库

#### 构建库

构建静态库`message`，它由两个文件构成。

```cmake
add_library(message
  STATIC
    Message.hpp
    Message.cpp
  )
```

`add_library`的第二个参数可以是以下值：

* **STATIC**：用于创建静态库，即编译文件的打包存档，以便在链接其他目标时使用，例如：可执行文件。
* **SHARED**：用于创建动态库，即可以动态链接，并在运行时加载的库。可以在`CMakeLists.txt`中使用`add_library(message SHARED
  Message.hpp Message.cpp) `从静态库切换到动态共享对象(DSO)。
* **OBJECT**：可将给定`add_library`的列表中的源码编译到目标文件，不将它们归档到静态库中，也不能将它们链接到共享对象中。**如果需要一次性创建静态库和动态库，那么使用对象库尤其有用。**
* **MODULE**：又为DSO组。与`SHARED`库不同，它们**不链接到项目中的任何目标，但是可以进行动态加载。**该参数可以用于构建运行时插件。

CMake还能够生成特殊类型的库，这不会在构建系统中产生输出，但是对于组织目标之间的依赖关系，和构建需求非常有用：

* **IMPORTED**：此类库目标表示位于项目外部的库。此类库的主要用途是，对现有依赖项进行构建。因此，`IMPORTED`库将被视为不可变的。参见: https://cmake.org/cmake/help/latest/manual/cmakebuildsystem.7.html#imported-targets 
* **INTERFACE**：与`IMPORTED`库类似。不过，该类型库可变，没有位置信息。它主要用于项目之外的目标构建使用。参见: https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#interface-libraries 
* **ALIAS**：顾名思义，这种库为项目中已存在的库目标定义别名。不过，不能为`IMPORTED`库选择别名。参见: https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#alias-libraries

#### 将库链接到可执行文件

```cmake
target_link_libraries(hello-world message)
```

#### OBJECT库的使用

```cmake
cmake_minimum_required(VERSION 3.5 FATAL_ERROR)
project(recipe-03 LANGUAGES CXX)

add_library(message-objs
	OBJECT
		Message.hpp
		Message.cpp
	)
	
# this is only needed for older compilers
# but doesn't hurt either to have it
# 保证编译的目标文件与生成位置无关。一般只在某些平台或较老的编译器上需要设置
set_target_properties(message-objs
	PROPERTIES
		POSITION_INDEPENDENT_CODE 1
	)
	
add_library(message-shared
	SHARED
		$<TARGET_OBJECTS:message-objs>
	)
	
add_library(message-static
	STATIC
		$<TARGET_OBJECTS:message-objs>
	)
	
add_executable(hello-world hello-world.cpp)

target_link_libraries(hello-world message-static)
```

注：引用对象库的生成器表达式语法:`$<TARGET_OBJECTS:message-objs> `。

生成器表达式是CMake在生成时(即配置之后)构造，用于生成特定于配置的构建输出。

### 指定编译器与构建类型

#### 指定编译器

CMake将语言的编译器存储在` CMAKE_<LANG>_COMPILER`变量中，其中`  <LANG> `是受支持的任何一种语言，例如`CMAKE_CXX_COMPILER`， `CMAKE_CC_COMPILER`

使用CLI中的`-D`选项指定编译器：

```shell
$ cmake -D CMAKE_CXX_COMPILER=clang++ ..
```

查看可用的编译器和编译器标志：

```shell
$ cmake --system-information information.txt
```

CMake提供了额外的变量来与编译器交互：

* `CMAKE_<LANG>_COMPILER_LOADED `:如果为项目启用了语言`<LANG>`，则将设置为`TRUE`。
* `CMAKE_<LANG>_COMPILER_ID`:编译器标识字符串，编译器供应商所特有。例如，`GCC`用于GNU编译器集合，`AppleClang`用于macOS上的Clang, `MSVC`用于Microsoft Visual Studio编译器。注意，不能保证为所有编译器或语言定义此变量。
* `CMAKE_COMPILER_IS_GNU<LANG> `:如果语言`<LANG>`是GNU编译器集合的一部分，则将此逻辑变量设置为`TRUE`。注意变量名的`<LANG>`部分遵循GNU约定：C语言为`CC`, C++语言为`CXX`, Fortran语言为`G77`。
* `CMAKE_<LANG>_COMPILER_VERSION`:此变量包含一个字符串，该字符串给定语言的编译器版本。版本信息在`major[.minor[.patch[.tweak]]]`中给出。但是，对于`CMAKE_<LANG>_COMPILER_ID`，不能保证所有编译器或语言都定义了此变量。

#### 构建类型

控制生成构建系统的配置变量是`CMAKE_BUILD_TYPE`。该变量默认为空，CMake识别的值为:

1. **Debug**：用于在没有优化的情况下，使用带有调试符号构建库或可执行文件。
2. **Release**：用于构建的优化的库或可执行文件，不包含调试符号。
3. **RelWithDebInfo**：用于构建较少的优化库或可执行文件，包含调试符号。
4. **MinSizeRel**：用于不增加目标代码大小的优化方式，来构建库或可执行文件。

切换构建类型

```shell
$ cmake -D CMAKE_BUILD_TYPE=Debug ..
```

### 用条件语句控制编译

希望能在两种不同行为之间切换：

1. 将` Message.hpp`和`Message.cpp`构建成一个库，然后将生成库链接到`hello-world`可执行文件中。
2. 直接构建生成executable.

- 引入逻辑变量`USE_LIBRARY`，值为`OFF`。并打印他的值

```cmake
set(USE_LIBRARY OFF)

message(STATUS "Compile sources into a library? ${USE_LIBRARY}")
```

- CMake中定义了全局变量`BUILD_SHARED_LIBS`，我们把它设置成OFF。这样在调用`add_library()`并省略参数时，会构建静态库

```cmake
set(BUILD_SHARED_LIBS OFF)
```

- 引入变量`_sources`，它包括`Message.hpp`和`Message.cpp`

```cmake
list(APPEND _sources Message.hpp Message.cpp)
```

- 写if-else

```cmake
if(USE_LIBRARY)
	# add_library will create a static library
	# since BUILD_SHARED_LIBS is OFF
	add_library(message ${_sources})
	add_executable(hello-world hello-world.cpp)
	target_link_libraries(hello-world message)
else()
	add_executable(hello-world hello-world.cpp ${_sources})
endif()
```



## Basic



## Headers

### Directory Paths

CMake syntax specifies mang varibles which is helpful to find directories.

| Variable                 | Info                                                         |
| ------------------------ | ------------------------------------------------------------ |
| CMAKE_SOURCE_DIR         | The root source directory                                    |
| CMAKE_CURRENT_SOURCE_DIR | The current source directory if using sub-projects and directories. |
| PROJECT_SOURCE_DIR       | The source directory of the current cmake project.           |
| CMAKE_BINARY_DIR         | The root binary / build directory. This is the directory where you ran the cmake command. |
| CMAKE_CURRENT_BINARY_DIR | The build directory you are currently in.                    |
| PROJECT_BINARY_DIR       | The build directory for the current project.                 |

### Source Files Variable

Creating a variable which includes the source files is helpful. For example:

```cmake
set(SOURCES
	src/a.cpp
	src/main.cpp
)
add_executable(${PROJECT_NAME} ${SOURCES})
```

Tips: For modern CMake it is NOT recommended to use a variable for sources. Instead it is typical to directly declare the sources in the add_xxx function.

这里很迷惑。他又说不推荐使用了

### Including Directories

When you have different include folders, you can make your compiler aware of them using the `target_include_directories()` [function](https://cmake.org/cmake/help/v3.0/command/target_include_directories.html). 

通俗讲就是指定头文件的搜索目录。如果项目中include自己定义的头文件，要么就写成相对路径；要么就写文件名，这时要指定搜索路径。

For example:

```cmake
target_include_directories(target
    PRIVATE
        ${PROJECT_SOURCE_DIR}/include
)
```

### Verbose Output

run command `make VERBOSE=1` to see the full output of the build.

## Static Library

### Different File types

- `.o`文件：目标文件。
- `.a`文件：archive 归档包，即静态库文件(static library)。其实质是多个`.o`文件打包的结果。
- `.so`文件：shared object 共享库/共享目标

CMakeLists结构如下：

```cmake
cmake_minimum_required(VERSION 3.5)

project(hello_library)

############################################################
# Create a library
############################################################

#Generate the static library from the library sources
add_library(hello_library STATIC 
    src/Hello.cpp
)

target_include_directories(hello_library
    PUBLIC 
        ${PROJECT_SOURCE_DIR}/include
)


############################################################
# Create an executable
############################################################

# Add an executable with the above sources
add_executable(hello_binary 
    src/main.cpp
)

# link the new hello_library target with the hello_binary target
target_link_libraries( hello_binary
    PRIVATE 
        hello_library
)
```

Firstly, use function `add_library()` and `target_include_directories()` to generate a static library. 

Then, use `target_link_libraries()` to link a static library. 



### Add a static library

`add_library()` function can create a library from some source files. 

"STATIC" means create a static library.

```cmake
add_library(hello_library STATIC
src/Hello.cpp
)
```

This will create a static library with the name libhello_library.a with the sources in the add_library call.

### Populating Including Directories

We include directories in the library using the `target_include_directories()` function with the scope set to PUBLIC

```cmake
target_include_directories(hello_library
    PUBLIC
        ${PROJECT_SOURCE_DIR}/include
)
```

This will cause the included directory used in the following places:

- When compiling the library
- When compiling any additional target that links the library.

The meaning of scopes are:

- PRIVATE - the directory is added to this target’s include directories
- INTERFACE - the directory is added to the include directories for any targets that link this library.
- PUBLIC - As above, it is included in this library and also any targets that link this library.

### Linking a Library

When creating an executable that will use your library, you must tell the complier about the library.

You can use `target_link_libraries()` function.

```cmake
add_executable(hello_binary
src/main.cpp
)
target_link_libraries(hello_binary
PRIVATE
	hello_library)
```

### PRIVATE INTERFACE PUBLIC



参考https://zhuanlan.zhihu.com/p/82244559

工程目录为：

```bash
cmake-test/                 工程主目录，main.c 调用 libhello-world.so
├── CMakeLists.txt
├── hello-world             生成 libhello-world.so，调用 libhello.so 和 libworld.so
│   ├── CMakeLists.txt
│   ├── hello               生成 libhello.so 
│   │   ├── CMakeLists.txt
│   │   ├── hello.c
│   │   └── hello.h         libhello.so 对外的头文件
│   ├── hello_world.c
│   ├── hello_world.h       libhello-world.so 对外的头文件
│   └── world               生成 libworld.so
│       ├── CMakeLists.txt
│       ├── world.c
│       └── world.h         libworld.so 对外的头文件
└── main.c
```

调用关系为：

```bash
                                 ├────libhello.so
可执行文件────libhello-world.so
                                 ├────libworld.so
```

- PRIAVTE

私有的。

- INTERFACE

接口

- PUBLIC

这个还没太搞懂

## Shared Library

工程目录：

```
├── CMakeLists.txt
├── include
│   └── shared
│       └── Hello.h
└── src
    ├── Hello.cpp
    └── main.cpp
```

CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.5)

project(hello_library)

############################################################
# Create a library
############################################################

#Generate the shared library from the library sources
add_library(hello_library SHARED 
    src/Hello.cpp
)
add_library(hello::library ALIAS hello_library)

target_include_directories(hello_library
    PUBLIC 
        ${PROJECT_SOURCE_DIR}/include
)

############################################################
# Create an executable
############################################################

# Add an executable with the above sources
add_executable(hello_binary
    src/main.cpp
)

# link the new hello_library target with the hello_binary target
target_link_libraries( hello_binary
    PRIVATE 
        hello::library
)
```

Firstly generate a shared library. Then add the executable.

### Add a shared library

Use function `add_library()`

```cmake
add_library(hello_library SHARED
    src/Hello.cpp
)
```

The will create a shared library named libhello_library.so.

### Alias Target

An alias target is an alternative name for a target.

```cmake
add_library(hello::library ALIAS hello_library)
```

### Linking a shared library

Use `target_link_libraries()` function.

```cmake
add_executable(hello_binary
    src/main.cpp
)

target_link_libraries(hello_binary
    PRIVATE
        hello::library
)
```





## 创建静态链接库和动态链接库

- `add_library(message STATIC Message.hpp Message.cpp)`：生成必要的构建指令，将指定的源码编译到库中。`add_library`的第一个参数是目标名。整个`CMakeLists.txt`中，可使用相同的名称来引用库。生成的库的实际名称将由CMake通过在前面添加前缀`lib`和适当的扩展名作为后缀来形成。生成库是根据第二个参数(`STATIC`或`SHARED`)和操作系统确定的。
- `target_link_libraries(hello-world message)`: 将库链接到可执行文件。此命令还确保`hello-world`可执行文件可以正确地依赖于消息库。因此，在消息库链接到`hello-world`可执行文件之前，需要完成消息库的构建。

编译成功后，构建目录包含`libmessage.a`一个静态库(在GNU/Linux上)和`hello-world`可执行文件。

CMake接受其他值作为`add_library`的第二个参数的有效值，我们来看下本书会用到的值：

- **STATIC**：用于创建静态库，即编译文件的打包存档，以便在链接其他目标时使用，例如：可执行文件。
- **SHARED**：用于创建动态库，即可以动态链接，并在运行时加载的库。可以在`CMakeLists.txt`中使用`add_library(message SHARED Message.hpp Message.cpp) `从静态库切换到动态共享对象(DSO)。
- **OBJECT**：可将给定`add_library`的列表中的源码编译到目标文件，不将它们归档到静态库中，也不能将它们链接到共享对象中。如果需要一次性创建静态库和动态库，那么使用对象库尤其有用。我们将在本示例中演示。
- **MODULE**：又为DSO组。与`SHARED`库不同，它们不链接到项目中的任何目标，不过可以进行动态加载。该参数可以用于构建运行时插件。

CMake还能够生成特殊类型的库，这不会在构建系统中产生输出，但是对于组织目标之间的依赖关系，和构建需求非常有用：

- **IMPORTED**：此类库目标表示位于项目外部的库。此类库的主要用途是，对现有依赖项进行构建。因此，`IMPORTED`库将被视为不可变的。我们将在本书的其他章节演示使用`IMPORTED`库的示例。参见: https://cmake.org/cmake/help/latest/manual/cmakebuildsystem.7.html#imported-targets
- **INTERFACE**：与`IMPORTED`库类似。不过，该类型库可变，没有位置信息。它主要用于项目之外的目标构建使用。参见: https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#interface-libraries
- **ALIAS**：顾名思义，这种库为项目中已存在的库目标定义别名。不过，不能为`IMPORTED`库选择别名。参见: https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#alias-libraries

