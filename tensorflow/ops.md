## Op定义分析

以tf.add为例，分析Op的定义过程。

```py
z = tf.add(x, y)
```
需要明确一点，上面的代码仅仅是在当前graph中添加一个节点，真正的计算在session执行时执行。而定义一个Op，主要分为三个步骤：
* 定义Op的接口
* 定义Op的计算逻辑OpKernel
* 定义Op的梯度


### Op接口的定义

TensorFlow内建Op接口定义在`tensorflow/core/ops/`目录下。Add操作的定义如下：

```cpp
// tensorflow/core/ops/math_ops.cc
REGISTER_OP("Add")
    .Input("x: T")
    .Input("y: T")
    .Output("z: T")
    .Attr(
        "T: {half, float, double, uint8, int8, int16, int32, int64, complex64, "
        "complex128, string}")
    .SetShapeFn(shape_inference::BroadcastBinaryOpShapeFn)
    .Doc(R"doc(
Returns x + y element-wise.

*NOTE*: `Add` supports broadcasting. `AddN` does not. More about broadcasting
[here](http://docs.scipy.org/doc/numpy/user/basics.broadcasting.html)
)doc");
```

主要注册了操作的接口，包括Python的docstring。其中比较重要的是定义并设置`ShapeFn`，用于确定输出的大小:

```cpp
// tensorflow/core/framework/common_shape_fns.cc
Status BroadcastBinaryOpShapeFn(InferenceContext* c) {
  ShapeHandle shape_x = c->input(0);
  ShapeHandle shape_y = c->input(1);
  if (!c->RankKnown(shape_x) || !c->RankKnown(shape_y)) {
    c->set_output(0, c->UnknownShape());
    return Status::OK();
  }

  // ...

  c->set_output(0, c->MakeShape(dims));
  return Status::OK();
}
```

注册的`ShapeFn`在session执行时被调用：

```cpp
    // tensorflow/core/common_runtime/shape_refiner.cc
    if (rerun_shape_fn) {
      // We have more information about the shapes on this pass,
      // so re-run shape inference.
      c->set_input_tensors(input_tensors);
      c->set_input_tensors_as_shapes(input_tensors_as_shapes);
      TF_RETURN_IF_ERROR(op_reg_data->shape_inference_fn(c.get()));
    }
```

`REGISTER_OP`定义如下：

```cpp
// tensorflow/core/framework/op.h
// Template specialization that turns all calls into no-ops.
template <>
class OpDefBuilderWrapper<false> {
 public:
  constexpr OpDefBuilderWrapper(const char name[]) {}
  OpDefBuilderWrapper<false>& Attr(StringPiece spec) { return *this; }
  OpDefBuilderWrapper<false>& Input(StringPiece spec) { return *this; }
  OpDefBuilderWrapper<false>& Output(StringPiece spec) { return *this; }
  OpDefBuilderWrapper<false>& SetIsCommutative() { return *this; }
  OpDefBuilderWrapper<false>& SetIsAggregate() { return *this; }
  OpDefBuilderWrapper<false>& SetIsStateful() { return *this; }
  OpDefBuilderWrapper<false>& SetAllowsUninitializedInput() { return *this; }
  OpDefBuilderWrapper<false>& Deprecated(int, StringPiece) { return *this; }
  OpDefBuilderWrapper<false>& Doc(StringPiece text) { return *this; }
  OpDefBuilderWrapper<false>& SetShapeFn(
      Status (*fn)(shape_inference::InferenceContext*)) {
    return *this;
  }
};

struct OpDefBuilderReceiver {
  // To call OpRegistry::Global()->Register(...), used by the
  // REGISTER_OP macro below.
  // Note: These are implicitly converting constructors.
  OpDefBuilderReceiver(
      const OpDefBuilderWrapper<true>& wrapper);  // NOLINT(runtime/explicit)
  constexpr OpDefBuilderReceiver(const OpDefBuilderWrapper<false>&) {
  }  // NOLINT(runtime/explicit)
};
}  // namespace register_op

#define REGISTER_OP(name) REGISTER_OP_UNIQ_HELPER(__COUNTER__, name)
#define REGISTER_OP_UNIQ_HELPER(ctr, name) REGISTER_OP_UNIQ(ctr, name)
#define REGISTER_OP_UNIQ(ctr, name)                                          \
  static ::tensorflow::register_op::OpDefBuilderReceiver register_op##ctr    \
      TF_ATTRIBUTE_UNUSED =                                                  \
          ::tensorflow::register_op::OpDefBuilderWrapper<SHOULD_REGISTER_OP( \
              name)>(name)

}  // namespace tensorflow
```

可以看到`REGISTER_OP`宏定义最终被展开成：

```cpp
OpDefBuilderReceiver register_op1234 __attribute__((unused)) = \
    OpDefBuilderWrapper(“Add”).Input("x: T").Input("y: T").Output("z: T").Attr(...).Doc(...);
```

最终Op被注册到了一个static变量`global_op_registry`中：

```cpp
// tensorflow/core/framework/op.cc
// static
OpRegistry* OpRegistry::Global() {
  static OpRegistry* global_op_registry = new OpRegistry;
  return global_op_registry;
}

namespace register_op {
OpDefBuilderReceiver::OpDefBuilderReceiver(
    const OpDefBuilderWrapper<true>& wrapper) {
  OpRegistry::Global()->Register(
      [wrapper](OpRegistrationData* op_reg_data) -> Status {
        return wrapper.builder().Finalize(op_reg_data);
      });
}
}  // namespace register_op
```

另外，从上面的代码可以看到，TensorFlow支持选择性的注册Op：

```cpp
// tensorflow/core/framework/selective_registration.h
#ifdef SELECTIVE_REGISTRATION

// Experimental selective registration support to reduce binary size.
//
// To use selective registration, when building:
// 1. define SELECTIVE_REGISTRATION, e.g. in gcc by passing
//    -DSELECTIVE_REGISTRATION to compilation.
// 2. Provide ops_to_register.h. This file is not included in the repo and must
//    be placed by the user or a tool where the compiler can find it.  It must
//    define the constants and functions used in the macros below. The
//    functions should be defined as valid constexpr functions, so that they are
//    evaluated at compile time: this is needed to make symbols referenced by
//    un-registered objects unused, and therefore allow the linker to strip them
//    out.  See tools/print_required_ops/print_selective_registration_header.py
//    for a tool that can be used to generate ops_to_register.h.
//
// ops_to_register.h should define macros for:
//   // Ops for which this is false will not be registered.
//   SHOULD_REGISTER_OP(op)
//   // If this is false, then no gradient ops are registered.
//   SHOULD_REGISTER_OP_GRADIENT
//   // Op kernel classes where this is false won't be registered.
//   SHOULD_REGISTER_OP_KERNEL(clz)
// The macros should be defined using constexprs.

#include "ops_to_register.h"

#if (!defined(SHOULD_REGISTER_OP) || !defined(SHOULD_REGISTER_OP_GRADIENT) || \
     !defined(SHOULD_REGISTER_OP_KERNEL))
static_assert(false, "ops_to_register.h must define SHOULD_REGISTER macros");
#endif
#else
#define SHOULD_REGISTER_OP(op) true
#define SHOULD_REGISTER_OP_GRADIENT true
#define SHOULD_REGISTER_OP_KERNEL(clz) true
#endif
```

**疑问**：bazel中链接kernel代码时并没有选择逻辑\(而且也没有办法选择性的编译\)，即全部都链接，而gcc\/ld的默认行为是包含所有符号\(参考[gcc删除未使用的函数](http://stackoverflow.com/questions/6687630/how-to-remove-unused-c-c-symbols-with-gcc-and-ld)\)。但bazel脚本中没有相关参数\(GPU编译中有但是与Android无关\)。

除自带的Op之外，TensorFlow支持用户自定义Op，方法可参考[Adding a New Op](https://www.tensorflow.org/versions/master/how_tos/adding_an_op/index.html)。

### Kernel的定义

Op真正的计算逻辑称为OpKernel，可以针对不同设备有不同的实现。内建的OpKernel目录在`tensorflow/core/kernels/`。

```cpp
// tensorflow/core/kernels/cwise_op_add_1.cc
REGISTER5(BinaryOp, CPU, "Add", functor::add, float, Eigen::half, double, int32,
          int64);

#if TENSORFLOW_USE_SYCL
#define REGISTER_SYCL_KERNEL(TYPE)                                    \
  REGISTER_KERNEL_BUILDER(                                            \
                          Name("Add")                                 \
                          .Device(DEVICE_SYCL)                        \
                          .TypeConstraint<TYPE>("T"),                 \
                          BinaryOp<SYCLDevice, functor::add<TYPE>>);
  REGISTER_SYCL_KERNEL(float);
#undef REGISTER_SYCL_KERNEL
#endif // TENSORFLOW_USE_SYCL

#if GOOGLE_CUDA
REGISTER3(BinaryOp, GPU, "Add", functor::add, float, Eigen::half, double);

// A special GPU kernel for int32.
// TODO(b/25387198): Also enable int32 in device memory. This kernel
// registration requires all int32 inputs and outputs to be in host memory.
REGISTER_KERNEL_BUILDER(Name("Add")
                            .Device(DEVICE_GPU)
                            .HostMemory("x")
                            .HostMemory("y")
                            .HostMemory("z")
                            .TypeConstraint<int32>("T"),
                        BinaryOp<CPUDevice, functor::add<int32>>);
#endif
```

```cpp
// tensorflow/core/kernels/cwise_op_add_2.cc
// REGISTER# macros ignore all but first type (assumed to be float) when
// __ANDROID_TYPES_SLIM__ is defined.  Since this file is the second of two
// sharded files, only make its register calls when not __ANDROID_TYPES_SLIM__.
#if !defined(__ANDROID_TYPES_SLIM__)

REGISTER5(BinaryOp, CPU, "Add", functor::add, int8, int16, complex64,
          complex128, string);
#if GOOGLE_CUDA
REGISTER3(BinaryOp, GPU, "Add", functor::add, int64, complex64, complex128);
#endif  // GOOGLE_CUDA

#endif  // !defined(__ANDROID_TYPES_SLIM__)
```

宏定义展开

```cpp
// tensorflow/core/kernels/cwise_ops_common.h
#define REGISTER(OP, D, N, F, T)                                             \
  REGISTER_KERNEL_BUILDER(Name(N).Device(DEVICE_##D).TypeConstraint<T>("T"), \
                          OP<D##Device, F<T>>);

// Macros to register kernels for multiple types (T0, T1, etc.)  on
// device type "D" (CPU or GPU) for operation "N" (e.g., sqrt) using
// the functor "F" (e.g., functor:sqrt).

#if defined(__ANDROID_TYPES_SLIM__)
// Note that __ANDROID_TYPES_SLIM__ is also checked in the cwise_ops*.cc files.
// Normally Android TensorFlow is built with a reduced number of types (float).
// Override on the command-line "--define ANDROID_TYPES=__ANDROID_TYPES_FULL__"
// to generate a library with full type support with a consequent increase in
// code size.
#define REGISTER2(OP, D, N, F, T0, T1) REGISTER(OP, D, N, F, T0)
#define REGISTER3(OP, D, N, F, T0, T1, T2) REGISTER(OP, D, N, F, T0)
#define REGISTER4(OP, D, N, F, T0, T1, T2, T3) REGISTER(OP, D, N, F, T0)
#define REGISTER5(OP, D, N, F, T0, T1, T2, T3, T4) REGISTER(OP, D, N, F, T0)
#define REGISTER6(OP, D, N, F, T0, T1, T2, T3, T4, T5) REGISTER(OP, D, N, F, T0)
#define REGISTER7(OP, D, N, F, T0, T1, T2, T3, T4, T5, T6) \
  REGISTER(OP, D, N, F, T0)
#define REGISTER8(OP, D, N, F, T0, T1, T2, T3, T4, T5, T6, T7) \
  REGISTER(OP, D, N, F, T0)
#define REGISTER9(OP, D, N, F, T0, T1, T2, T3, T4, T5, T6, T7, T8) \
  REGISTER(OP, D, N, F, T0)
#else  // !defined(__ANDROID_TYPES_SLIM__)
#define REGISTER2(OP, D, N, F, T0, T1) \
  REGISTER(OP, D, N, F, T0)            \
  REGISTER(OP, D, N, F, T1)
#define REGISTER3(OP, D, N, F, T0, T1, T2) \
  REGISTER2(OP, D, N, F, T0, T1)           \
  REGISTER(OP, D, N, F, T2)
#define REGISTER4(OP, D, N, F, T0, T1, T2, T3) \
  REGISTER2(OP, D, N, F, T0, T1)               \
  REGISTER2(OP, D, N, F, T2, T3)
#define REGISTER5(OP, D, N, F, T0, T1, T2, T3, T4) \
  REGISTER3(OP, D, N, F, T0, T1, T2)               \
  REGISTER2(OP, D, N, F, T3, T4)
#define REGISTER6(OP, D, N, F, T0, T1, T2, T3, T4, T5) \
  REGISTER3(OP, D, N, F, T0, T1, T2)                   \
  REGISTER3(OP, D, N, F, T3, T4, T5)
#define REGISTER7(OP, D, N, F, T0, T1, T2, T3, T4, T5, T6) \
  REGISTER4(OP, D, N, F, T0, T1, T2, T3)                   \
  REGISTER3(OP, D, N, F, T4, T5, T6)
#define REGISTER8(OP, D, N, F, T0, T1, T2, T3, T4, T5, T6, T7) \
  REGISTER4(OP, D, N, F, T0, T1, T2, T3)                       \
  REGISTER4(OP, D, N, F, T4, T5, T6, T7)
#define REGISTER9(OP, D, N, F, T0, T1, T2, T3, T4, T5, T6, T7, T8) \
  REGISTER5(OP, D, N, F, T0, T1, T2, T3, T4)                       \
  REGISTER4(OP, D, N, F, T5, T6, T7, T8)

// Instead of adding REGISTER10, etc., shard the .cc files - see
// cwise_op_equal_to_*.cc for an example.

#endif  // defined(__ANDROID_TYPES_SLIM__)
```

具体的计算方法，以`BinaryOp<CPUDevice, functor::add<int32>>`为例：

```cpp
// tensorflow/core/kernels/cwise_ops_common.h
template <typename Device, typename Functor>
class BinaryOp : public BinaryOpShared {
 public:
  // ...
  void Compute(OpKernelContext* ctx) override {
    // ...
        // scalar op tensor
        functor::BinaryFunctor<Device, Functor, 1>().Left(
            eigen_device, out_flat, in0.template scalar<Tin>(),
            in1.template flat<Tin>(), error_ptr);
    // ...
  }
```

可以看到，最终调用到了Eigen库，而Eigen同时支持CPU\/GPU计算。而对于复杂的DNN操作，GPU实现则调用了cuDNN的实现，对cuDNN的封装在`tensorflow/stream_executor`目录，其中SO的加载在`tensorflow/stream_executor/dso_loader.cc`中实现。

## 梯度的计算与注册

一个Op的梯度计算也是通过注册的方式添加的：

```py
# tensorflow/python/ops/math_grad.py
@ops.RegisterGradient("Add")
def _AddGrad(op, grad):
  x = op.inputs[0]
  y = op.inputs[1]
  sx = array_ops.shape(x)
  sy = array_ops.shape(y)
  rx, ry = gen_array_ops._broadcast_gradient_args(sx, sy)
  return (array_ops.reshape(math_ops.reduce_sum(grad, rx), sx),
          array_ops.reshape(math_ops.reduce_sum(grad, ry), sy))
```

注册过程

```py
# tensorflow/python/framework/ops.py
_gradient_registry = registry.Registry("gradient")

class RegisterGradient(object):
  def __call__(self, f):
    """Registers the function `f` as gradient function for `op_type`."""
    _gradient_registry.register(f, self._op_type)
    return f
```

Registry的定义在`tensorflow/python/framework/registry.py`，为纯Python实现。

另一种方式为C++实现：

```cpp
// tensorflow/core/ops/math_grad.cc
typedef FunctionDefHelper FDH;

// Cwise binary ops
Status GradForUnaryCwise(FunctionDef* g, std::vector<FDH::Node> nodes) {
  for (auto& n : nodes) {
    if (n.attr.empty()) {
      n.attr = {{"T", "$T"}};
    }
  }
  *g = FDH::Define(
      // Arg defs
      {"x: T", "dy: T"},
      // Ret val defs
      {"dx: T"},
      // Attr defs
      {{"T: {half, float, double}"}},
      // Nodes
      nodes);
  return Status::OK();
}

Status AbsGrad(const AttrSlice& attrs, FunctionDef* g) {
  // clang-format off
  return GradForUnaryCwise(g, {
      {{"sign"}, "Sign", {"x"}, {}, {"dy"}},
      {{"dx"}, "Mul", {"dy", "sign"}},
  });
  // clang-format on
}
REGISTER_OP_GRADIENT("Abs", AbsGrad);
```

**Q** C++代码也是基于基本的算子实现\(如Abs的梯度使用Sign, Mul两个算子\)，似乎两者并无性能上的差异，为何有两种实现方式？

**A** 向Vijay求证过，梯度实现向C++层面迁移，主要是为了能支持更多的语言做训练，另外一个原因是"There's experimental work to try to introduce 'functions' as a first class primitive in the GraphDef, and SymbolicGradient is used to be able to differentiate functions"

下面是调用的区别\(Python代码中也有分开处理\)：

```cpp
// tensorflow/core/common_runtime/function.cc

Status FunctionLibraryRuntimeImpl::InstantiateSymbolicGradient(
    const NameAttrList& func, FunctionBody** g_body) {
  const FunctionDef* fdef = lib_def_->Find(func.name());
  if (fdef == nullptr) {
    // f is a primitive op.
    gradient::Creator creator;
    TF_RETURN_IF_ERROR(gradient::GetOpGradientCreator(func.name(), &creator));
    if (creator == nullptr) {
      return errors::InvalidArgument("No gradient is defined for ",
                                     func.name());
    }
    FunctionDef grad_fdef;
    // TODO(josh11b): Should filter out the attrs from func that aren't used
    // by the gradient function.
    TF_RETURN_IF_ERROR(creator(AttrSlice(&func.attr()), &grad_fdef));
    TF_RETURN_IF_ERROR(FunctionDefToBody(grad_fdef, func.attr(), g_body));
  } else {
    // f is a user-defined function.
    Handle f_handle;
    TF_RETURN_IF_ERROR(Instantiate(func.name(), func.attr(), &f_handle));
    const FunctionBody* f_body = GetFunctionBody(f_handle);
    CHECK_NOTNULL(f_body);
    *g_body = SymbolicGradient(*f_body);
  }
  return Status::OK();
}
```

## Python层代码的生成

最后分析一下，上面的定义是如何跟Python代码结合起来的。首先看一下Python的调用方式

```
import tensorflow as tf
z = tf.add(x, y)
```

tensorflow的导入

```py
# tensorflow/python/__init__.py
from tensorflow.python.ops import math_ops
```

math\_ops的导入

```py
# tensorflow/python/ops/math_ops.py
from tensorflow.python.ops import gen_math_ops
```

gen\_math\_ops中的定义

```py
# path/to/install/site-packages/tensorflow/python/ops/gen_math_ops.py
_add_outputs = ["z"]


def add(x, y, name=None):
  r"""Returns x + y element-wise.

  *NOTE*: Add supports broadcasting. AddN does not.

  Args:
    x: A `Tensor`. Must be one of the following types: `half`, `float32`, `float64`, `uint8`, `int8`, `int16`, `int32`, `int64`, `complex64`, `complex128`, `string`.
    y: A `Tensor`. Must have the same type as `x`.
    name: A name for the operation (optional).

  Returns:
    A `Tensor`. Has the same type as `x`.
  """
  result = _op_def_lib.apply_op("Add", x=x, y=y, name=name)
  return result
```

`_op_def_lib.apply_op`的作用是在graph中添加对应名字的Op节点。需要注意的是，Op的梯度计算节点并不是在这里加入到graph中的，这里仅仅加入了前向计算节点。其中，gen\_math\_ops.py文件是在bazel构建过程中生成的：

```py
# tensorflow/python/BUILD
tf_gen_op_wrapper_private_py(
    name = "math_ops_gen",
    require_shape_functions = True,
)
```

tf\_gen\_op\_wrapper\_private\_py的定义

```py
# tensorflow/python/build_defs.bzl
def tf_gen_op_wrapper_private_py(name, out=None, deps=[],
                                 require_shape_functions=False):
  if not name.endswith("_gen"):
    fail("name must end in _gen")
  bare_op_name = name[:-4] # Strip of the _gen
  tf_gen_op_wrapper_py(name=bare_op_name,
    out=out,
    hidden_file="ops/hidden_ops.txt",
    visibility=["//visibility:private"],
    deps=deps,
    require_shape_functions=require_shape_functions,
    generated_target_name=name,
  )
```
```py
    # tensorflow/tensorflow.bzl
    def tf_gen_op_wrapper_py(name, out=None, hidden=None, visibility=None, deps=[],
                             require_shape_functions=False, hidden_file=None,
                             generated_target_name=None):
      # Construct a cc_binary containing the specified ops.
      tool_name = "gen_" + name + "_py_wrappers_cc"
      if not deps:
        deps = ["//tensorflow/core:" + name + "_op_lib"]
      native.cc_binary(
          name = tool_name,
          linkopts = ["-lm"],
          copts = tf_copts(),
          linkstatic = 1,   # Faster to link this one-time-use binary dynamically
          deps = (["//tensorflow/core:framework",
                   "//tensorflow/python:python_op_gen_main"] + deps),
          visibility = ["//tensorflow:internal"],
      )

      # Invoke the previous cc_binary to generate a python file.
      if not out:
        out = "ops/gen_" + name + ".py"

      if hidden:
        # `hidden` is a list of op names to be hidden in the generated module.
        native.genrule(
            name=name + "_pygenrule",
            outs=[out],
            tools=[tool_name],
            cmd=("$(location " + tool_name + ") " + ",".join(hidden)
                 + " " + ("1" if require_shape_functions else "0") + " > $@"))
      elif hidden_file:
        # `hidden_file` is file containing a list of op names to be hidden in the
        # generated module.
        native.genrule(
            name=name + "_pygenrule",
            outs=[out],
            srcs=[hidden_file],
            tools=[tool_name],
            cmd=("$(location " + tool_name + ") @$(location "
                 + hidden_file + ") " + ("1" if require_shape_functions else "0")
                 + " > $@"))
      else:
        # No ops should be hidden in the generated module.
        native.genrule(
            name=name + "_pygenrule",
            outs=[out],
            tools=[tool_name],
            cmd=("$(location " + tool_name + ") "
                 + ("1" if require_shape_functions else "0") + " > $@"))

      # Make a py_library out of the generated python file.
      if not generated_target_name:
        generated_target_name = name
      native.py_library(name=generated_target_name,
                        srcs=[out],
                        srcs_version="PY2AND3",
                        visibility=visibility,
                        deps=[
                            "//tensorflow/python:framework_for_generated_wrappers",
                        ],)
```
从bazel规则中可以看到，对应的python文件是通过`python_op_gen_main`工具生成的，其定义如下：

```cpp
// tensorflow/python/framework/python_op_gen_main.cc
void PrintAllPythonOps(const std::vector<string>& hidden_ops,
                       bool require_shapes) {
  OpList ops;
  OpRegistry::Global()->Export(false, &ops);
  PrintPythonOps(ops, hidden_ops, require_shapes);
}

// ...

int main(int argc, char* argv[]) {
  tensorflow::port::InitMain(argv[0], &argc, &argv);

  // Usage:
  //   gen_main [ @FILENAME | OpName[,OpName]* ] (0 | 1)
  if (argc == 2) {
    tensorflow::PrintAllPythonOps({}, tensorflow::string(argv[1]) == "1");
  } else if (argc == 3) {
    std::vector<tensorflow::string> hidden_ops;
    TF_CHECK_OK(tensorflow::ParseHiddenOpsCommandLine(argv[1], &hidden_ops));
    tensorflow::PrintAllPythonOps(hidden_ops,
                                  tensorflow::string(argv[2]) == "1");
  } else {
    return -1;
  }
  return 0;
}
```

可以看到，`python_op_gen_main`工具通过链接对应的lib，得到对应的`OpRegistry`，从而生成对应的`gen_xx_ops.py`文件。

