# cmake与brew

用brew安装cmake:

```
brew install cmake
```

用brew安装boost:

```
brew install boost
```

安装的位置是`/opt/homebrew/Cellar/boost/1.76.0/`。可以先通过`printenv`看一下brew的位置，然后在找。

如果要使用boost，需要在CMakeLists.txt中写入：

```
include_directories(/opt/homebrew/Cellar/boost/1.76.0/include)
link_directories(/opt/homebrew/Cellar/boost/1.76.0/lib)
```

