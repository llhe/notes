## RPC

### bfloat16压缩
为减少数据传输(不仅用于RPC，还包括PCIe传输)，将flaot32压缩成2字节。方法是简单的取前两字节，计算高效。
```cpp
// tensorflow/core/framework/bfloat16.h
// Compact 16-bit encoding of floating point numbers. This representation uses
// 1 bit for the sign, 8 bits for the exponent and 7 bits for the mantissa.  It
// is assumed that floats are in IEEE 754 format so the representation is just
// bits 16-31 of a single precision float.
//
// NOTE: The IEEE floating point standard defines a float16 format that
// is different than this format (it has fewer bits of exponent and more
// bits of mantissa).  We don't use that format here because conversion
// to/from 32-bit floats is more complex for that format, and the
// conversion for this format is very simple.
//
// Because of the existing IEEE float16 type, we do not name our representation
// "float16" but just use "uint16".
//
// <-----our 16bits float------->
// s e e e e e e e e f f f f f f f f f f f f f f f f f f f f f f f
// <------------------------------float-------------------------->
// 3 3             2 2             1 1                           0
// 1 0             3 2             5 4                           0
//
//
// This type only supports conversion back and forth with float.
//
// This file must be compilable by nvcc.
//
// The type is defined in framework/numeric_types.h.

namespace tensorflow {

// Conversion routines between an array of float and bfloat16 of
// "size".
void FloatToBFloat16(const float* src, bfloat16* dst, int64 size);
void BFloat16ToFloat(const bfloat16* src, float* dst, int64 size);

}  // namespace tensorflow
```
```cpp
// tensorflow/core/framework/bfloat16.cc
// 问题：计算是否能够内联？否则效率有问题
// http://stackoverflow.com/questions/9338152/must-the-definition-of-a-c-inline-functions-be-in-the-same-file
// Link Time Optimization (LTO): https://gcc.gnu.org/wiki/LinkTimeOptimization
// 但目前TF的build文件并无次选项
void FloatToBFloat16(const float* src, bfloat16* dst, int64 size) {
  const uint16_t* p = reinterpret_cast<const uint16_t*>(src);
  uint16_t* q = reinterpret_cast<uint16_t*>(dst);
  for (; size; p += 2, q++, size--) {
    *q = p[1];
  }
}

void BFloat16ToFloat(const bfloat16* src, float* dst, int64 size) {
  const uint16_t* p = reinterpret_cast<const uint16_t*>(src);
  uint16_t* q = reinterpret_cast<uint16_t*>(dst);
  for (; size; p++, q += 2, size--) {
    q[0] = 0;
    q[1] = *p;
  }
}
```

### Tensor传输优化
采用protobuf repeated方式传输tensor，会带来较大的序列化开销：
* 大小端归一化
* 变长编码

而这些操作对训练过程并无帮助，TF代码采用了直接传输tensor的bytes方式，将序列化的开销降低到最小。

```cpp
// tensorflow/core/distributed_runtime/rpc/grpc_tensor_coding.cc
void EncodeTensorToByteBuffer(bool is_dead, const Tensor& val,
                              ::grpc::ByteBuffer* result) {
  if (!DataTypeCanUseMemcpy(val.dtype())) {
    // Straightforward but slow path for complicated kinds of tensor data
    // TODO(jeff,sanjay): If this becomes an issue, we could
    // go directly from val -> ByteBuffer, with some effort.
    val.AsProtoTensorContent(response.mutable_tensor());

    // Encode full protocol buffer to a ByteBuffer
    EncodeRecvTensorResponseToByteBuffer(response, result);
  } else {
    // skeleton is the encoded TensorProto contents (dtype and shape), but
    // not the actual data
    // 此处手动最各个field做了encoding，且对large tensor优化掉了memcpy，直接将buffer
    // 传递到了grpc中
}
```
GPU上的tensor处理路径有所不同
```cpp
                // TODO (jeff,sanjay,mrry): Avoid copy on GPU path by
                // modifying GPUUtil::SetProtoFromGPU to accept a
                // ::grpc::ByteBuffer to serialize to, rather than
                // encoding into a protocol buffer and then
                // serializing that (i.e. figure out how to use
                // EncodeTensorToByteBuffer on this path rather than
                // EncodeRecvTensorResponseToByteBuffer)
                GPUUtil::SetProtoFromGPU(val, src_dev, send_dev_context,
                                         tmp->mutable_tensor(), is_dead,
                                         response_ready);
```

### 传输聚合
TODO