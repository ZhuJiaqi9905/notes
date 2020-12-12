# CMake

## Basic

### CMakeLists.txt

`CMakeLists.txt` is a file which should store all your CMake commands.

CMake语言不区分大小写，但是参数区分大小写。

### Minimum CMake version

You should specify the minimum CMake version of a project use 

```cmake
cmake_minimum_required(VERSION 3.5)
```

### Projects

A name to make referencing certain variables easier **when using multiple projects**.

CXX means C++

```cmake
project(hello_cmake LANGUAGES CXX)
```

### Creating an Executable

Use `add_executable()` command to specify that an executable should be build from the specified source files. The first argument is the name of the executable and the second argument is the list of source files to compile which use whitespace or line break to split.

```
add_executable(hello_cmake main.cpp)
```

You can also use variables to output a executable.

```
cmake_minimum_required(VERSION 2.6)
project (hello_cmake)
add_executable(${PROJECT_NAME} main.cpp)
```

**The `project()` function, will create a variable ${PROJECT_NAME} **with the value hello_cmake. This can then be passed to the `add_executable()` function to output a 'hello_cmake' executable.

### Binary Directory

- CMAKE_BINARY_DIR : the root folder for all your binary files. It is  the root or top level that you run your cmake command from.
- CMake supports building and generating binary files with
  - in-place
  - out-of-source

#### In-Place Build

-  generate all temporary build files in the same directory structure as the source code
- run the `cmake .` command in your root directory

#### Out-of-Source Build

- allow you to create a single build folder that can be anywhere on your file system
- can keep your source tree clean

- To build, you need to 
  - change directory to the folder that you want to generate all temporary build files in.
  - run command `cmake fileDir` , the fileDir is where the CMakeLists.txt exists. You can use relative directory

After running `cmake`，you will find a MakeFile file，then you can run the command `make` to get an executable.

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

## 用条件语句控制编译

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

