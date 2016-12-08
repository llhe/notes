## 参数训练与梯度反向传播过程

TensorFlow支持[自动梯度计算](https://en.wikipedia.org/wiki/Automatic_differentiation)。而从[Op定义](./ops.md)的分析过程可以看到，使用`tf.add`这样的代码，并不会在图中自动加入梯度计算节点。实际上，梯度反向传播的计算节点是在调用Optimizer.minimize时加入的。

```py
# tensorflow/python/training/optimizer.py
class Optimizer(object):
  """Base class for optimizers.

  This class defines the API to add Ops to train a model.  You never use this
  class directly, but instead instantiate one of its subclasses such as
  `GradientDescentOptimizer`, `AdagradOptimizer`, or `MomentumOptimizer`.

  ## Usage

  ###python
  # Create an optimizer with the desired parameters.
  opt = GradientDescentOptimizer(learning_rate=0.1)
  # Add Ops to the graph to minimize a cost by updating a list of variables.
  # "cost" is a Tensor, and the list of variables contains tf.Variable
  # objects.
  opt_op = opt.minimize(cost, var_list=<list of variables>)
  ###

  In the training program you will just have to run the returned Op.

  ###python
  # Execute opt_op to do one step of training:
  opt_op.run()
  ###

  ### Processing gradients before applying them.

  Calling `minimize()` takes care of both computing the gradients and
  applying them to the variables.  If you want to process the gradients
  before applying them you can instead use the optimizer in three steps:

  1.  Compute the gradients with `compute_gradients()`.
  2.  Process the gradients as you wish.
  3.  Apply the processed gradients with `apply_gradients()`.

  Example:

  ###
  # Create an optimizer.
  opt = GradientDescentOptimizer(learning_rate=0.1)

  # Compute the gradients for a list of variables.
  grads_and_vars = opt.compute_gradients(loss, <list of variables>)

  # grads_and_vars is a list of tuples (gradient, variable).  Do whatever you
  # need to the 'gradient' part, for example cap them, etc.
  capped_grads_and_vars = [(MyCapper(gv[0]), gv[1]) for gv in grads_and_vars]

  # Ask the optimizer to apply the capped gradients.
  opt.apply_gradients(capped_grads_and_vars)
  ###

  @@__init__

  @@minimize
  @@compute_gradients
  @@apply_gradients

  ### Gating Gradients

  Both `minimize()` and `compute_gradients()` accept a `gate_gradients`
  argument that controls the degree of parallelism during the application of
  the gradients.

  The possible values are: `GATE_NONE`, `GATE_OP`, and `GATE_GRAPH`.

  <b>`GATE_NONE`</b>: Compute and apply gradients in parallel.  This provides
  the maximum parallelism in execution, at the cost of some non-reproducibility
  in the results.  For example the two gradients of `matmul` depend on the input
  values: With `GATE_NONE` one of the gradients could be applied to one of the
  inputs _before_ the other gradient is computed resulting in non-reproducible
  results.

  <b>`GATE_OP`</b>: For each Op, make sure all gradients are computed before
  they are used.  This prevents race conditions for Ops that generate gradients
  for multiple inputs where the gradients depend on the inputs.

  <b>`GATE_GRAPH`</b>: Make sure all gradients for all variables are computed
  before any one of them is used.  This provides the least parallelism but can
  be useful if you want to process all gradients before applying any of them.

  ### Slots

  Some optimizer subclasses, such as `MomentumOptimizer` and `AdagradOptimizer`
  allocate and manage additional variables associated with the variables to
  train.  These are called <i>Slots</i>.  Slots have names and you can ask the
  optimizer for the names of the slots that it uses.  Once you have a slot name
  you can ask the optimizer for the variable it created to hold the slot value.

  This can be useful if you want to log debug a training algorithm, report stats
  about the slots, etc.

  @@get_slot_names
  @@get_slot
  """
```

从注释中可以看到`minimize`主要有两个步骤
* 计算梯度
* 更新梯度

实际上，这里只是在图中加入节点，而非真正进行计算。对照`minimize`代码
```py
# tensorflow/python/training/optimizer.py
  def minimize(self, loss, global_step=None, var_list=None,
               gate_gradients=GATE_OP, aggregation_method=None,
               colocate_gradients_with_ops=False, name=None,
               grad_loss=None):
    """Add operations to minimize `loss` by updating `var_list`.

    This method simply combines calls `compute_gradients()` and
    `apply_gradients()`. If you want to process the gradient before applying
    them call `compute_gradients()` and `apply_gradients()` explicitly instead
    of using this function.

    Args:
      loss: A `Tensor` containing the value to minimize.
      global_step: Optional `Variable` to increment by one after the
        variables have been updated.
      var_list: Optional list of `Variable` objects to update to minimize
        `loss`.  Defaults to the list of variables collected in the graph
        under the key `GraphKeys.TRAINABLE_VARIABLES`.
      gate_gradients: How to gate the computation of gradients.  Can be
        `GATE_NONE`, `GATE_OP`, or  `GATE_GRAPH`.
      aggregation_method: Specifies the method used to combine gradient terms.
        Valid values are defined in the class `AggregationMethod`.
      colocate_gradients_with_ops: If True, try colocating gradients with
        the corresponding op.
      name: Optional name for the returned operation.
      grad_loss: Optional. A `Tensor` holding the gradient computed for `loss`.

    Returns:
      An Operation that updates the variables in `var_list`.  If `global_step`
      was not `None`, that operation also increments `global_step`.

    Raises:
      ValueError: If some of the variables are not `Variable` objects.
    """
    grads_and_vars = self.compute_gradients(
        loss, var_list=var_list, gate_gradients=gate_gradients,
        aggregation_method=aggregation_method,
        colocate_gradients_with_ops=colocate_gradients_with_ops,
        grad_loss=grad_loss)

    vars_with_grad = [v for g, v in grads_and_vars if g is not None]
    if not vars_with_grad:
      raise ValueError(
          "No gradients provided for any variable, check your graph for ops"
          " that do not support gradients, between variables %s and loss %s." %
          ([str(v) for _, v in grads_and_vars], loss))

    return self.apply_gradients(grads_and_vars, global_step=global_step,
                                name=name)
```

从注释中可以看到，梯度的计算方式根据并发度和一致性的不同，有三种组织形式
* GATE_NONE
* GATE_OP
* GATE_GRAPH

此处代码均为python实现，并无太多trick，不再详细分析，最后看一下，定义Op时注册的梯度是如何使用的。
```py
# tensorflow/python/ops/gradients_impl.py
def gradients(ys,
              xs,
              grad_ys=None,
              name="gradients",
              colocate_gradients_with_ops=False,
              gate_gradients=False,
              aggregation_method=None):
  """Constructs symbolic partial derivatives of sum of `ys` w.r.t. x in `xs`.

  `ys` and `xs` are each a `Tensor` or a list of tensors.  `grad_ys`
  is a list of `Tensor`, holding the gradients received by the
  `ys`. The list must be the same length as `ys`.

  `gradients()` adds ops to the graph to output the partial
  derivatives of `ys` with respect to `xs`.  It returns a list of
  `Tensor` of length `len(xs)` where each tensor is the `sum(dy/dx)`
  for y in `ys`.

  `grad_ys` is a list of tensors of the same length as `ys` that holds
  the initial gradients for each y in `ys`.  When `grad_ys` is None,
  we fill in a tensor of '1's of the shape of y for each y in `ys`.  A
  user can provide their own initial `grad_ys` to compute the
  derivatives using a different initial gradient for each y (e.g., if
  one wanted to weight the gradient differently for each value in
  each y).

  Args:
    ys: A `Tensor` or list of tensors to be differentiated.
    xs: A `Tensor` or list of tensors to be used for differentiation.
    grad_ys: Optional. A `Tensor` or list of tensors the same size as
      `ys` and holding the gradients computed for each y in `ys`.
    name: Optional name to use for grouping all the gradient ops together.
      defaults to 'gradients'.
    colocate_gradients_with_ops: If True, try colocating gradients with
      the corresponding op.
    gate_gradients: If True, add a tuple around the gradients returned
      for an operations.  This avoids some race conditions.
    aggregation_method: Specifies the method used to combine gradient terms.
      Accepted values are constants defined in the class `AggregationMethod`.

  Returns:
    A list of `sum(dy/dx)` for each x in `xs`.

  Raises:
    LookupError: if one of the operations between `x` and `y` does not
      have a registered gradient function.
    ValueError: if the arguments are invalid.

  """
  # ...
        grad_fn = None
        # pylint: disable=protected-access
        is_func_call = ops.get_default_graph()._is_function(op.type)
        has_out_grads = any(isinstance(g, ops.Tensor) or g for g in out_grads)
        if has_out_grads and (op._id not in stop_ops):
          if is_func_call:
            grad_fn = ops.get_default_graph()._get_function(
                op.type).python_grad_func
            # pylint: enable=protected-access
          else:
            # A grad_fn must be defined, either as a function or as None
            # for ops that do not have gradients.
            try:
              grad_fn = ops.get_gradient_function(op)
            except LookupError:
              raise LookupError(
                  "No gradient defined for operation '%s' (op type: %s)" %
                  (op.name, op.type))
  # ...
          with ops.name_scope(op.name + "_grad"):
            # pylint: disable=protected-access
            with ops.get_default_graph()._original_op(op):
              # pylint: enable=protected-access
              if grad_fn:
                # If grad_fn was found, do not use SymbolicGradient even for
                # functions.
                in_grads = grad_fn(op, *out_grads)
              else:
                # For function call ops, we add a 'SymbolicGradient'
                # node to the graph to compute gradients.
                in_grads = _SymGrad(op, out_grads)
```
从代码中看到，`grad_fn = ops.get_gradient_function(op)`从注册表中查找注册的梯度计算方法，而`in_grads = grad_fn(op, *out_grads)`则执行了这段代码。现在回顾一下梯度的注册
```py
# tensorflow/python/ops/math_grad.py
@ops.RegisterGradient("Abs")
def _AbsGrad(op, grad):
  x = op.inputs[0]
  return grad * math_ops.sign(x)
```
可以看到，注册的方法也只是普通的tensorflow计算图定义。也就是说，对于底层而言，梯度计算和反向传播过程跟正向计算没有任何区别。

TODO: is_func_call分支对应C++中注册的梯度
