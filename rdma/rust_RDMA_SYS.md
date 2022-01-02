# rust: RDMA_SYS

是用rust把rdma的结构体和函数都包装了一层。

- 大部分是使用bindgen库自动生成的代码
  - 在`bindings.rs`中
- 但是有一些C函数是inline的，并且结构体是未命名的union。针对这些函数和结构体，作者单独进行了封装。
  - 在	`types.rs`和`verbs.rs`

- 版本要求：

```
const LIB_IBVERBS_DEV_VERSION: &str = "1.8.28";
const LIB_RDMACM_DEV_VERSION: &str = "1.2.28";
```

