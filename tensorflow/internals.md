## 源码分析
TensorFlow主要模块的源码解读(基于commit 552cdf68bdc58ac0a27756975c800084fd443300)。

```
.
├── c
├── cc
├── contrib
├── core
│   ├── common_runtime 公共运行时环境以及本地运行时实现
│   ├── debug
│   ├── distributed_runtime 分布式运行时实现
│   ├── example 标准特征格式定义
│   ├── framework 基础数据结构定义
│   ├── graph 图相关定义与算法
│   ├── kernels 内建算子的实现
│   ├── lib 公共基础库
│   ├── ops 内建算子的接口定义
│   ├── platform 平台相关代码的封装(锁，文件系统等)
│   ├── protobuf 公开的proto接口定义，跨语言/RPC传输
│   ├── public 公开接口，主要是session和版本
│   ├── user_ops
│   └── util
├── examples
├── g3doc
├── go
├── models
├── python
│   ├── client
│   ├── debug
│   ├── framework
│   ├── kernel_tests
│   ├── layers
│   ├── lib
│   ├── ops 基础op/梯度定义，其中包含了对C++ Op的封装
│   ├── platform
│   ├── saved_model
│   ├── summary
│   ├── tools
│   ├── training 梯度计算与训练算法
│   ├── user_ops
│   └── util
├── stream_executor Google的并行计算库(https://github.com/henline/streamexecutordoc)
│   └── cuda CUDA封装
├── tensorboard
├── tools
│   ├── benchmark
│   ├── ci_build
│   ├── dist_test
│   ├── docker
│   ├── docs
│   ├── gcs_test
│   ├── git
│   ├── graph_transforms
│   ├── pip_package
│   ├── proto_text
│   ├── quantization
│   ├── swig
│   ├── test
│   └── tfprof
└── user_ops
```

