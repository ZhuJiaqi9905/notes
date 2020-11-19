# CMake

## Introduction

### CMakeLists.txt

`CMakeLists.txt` is a file which should store all your CMake commands.

### Minimum CMake version

You should specify the minimum CMake version of a project use 

```cmake
cmake_minimum_required(VERSION 3.5)
```

### Projects

A name to make referencing certain variables easier **when using multiple projects**.

```cmake
project(hello_cmake)
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
- run the cmake command in your root directory

#### Out-of-Source Build

- allow you to create a single build folder that can be anywhere on your file system
- can keep your source tree clean

- To build, you need to 
  - change directory to the folder that you want to generate all temporary build files in.
  - run command `make fileDir` , the fileDir is where the CMakeLists.txt exists.