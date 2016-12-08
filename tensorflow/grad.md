## 梯度正确性验证

TODO
```py
# tensorflow/python/ops/gradient_checker.py
def compute_gradient_error(x,
                           x_shape,
                           y,
                           y_shape,
                           x_init_value=None,
                           delta=1e-3,
                           init_targets=None,
                           extra_feed_dict=None):
  """Computes the gradient error.

  Computes the maximum error for dy/dx between the computed Jacobian and the
  numerically estimated Jacobian.

  This function will modify the tensors passed in as it adds more operations
  and hence changing the consumers of the operations of the input tensors.

  This function adds operations to the current session. To compute the error
  using a particular device, such as a GPU, use the standard methods for
  setting a device (e.g. using with sess.graph.device() or setting a device
  function in the session constructor).

  Args:
    x: a tensor or list of tensors
    x_shape: the dimensions of x as a tuple or an array of ints. If x is a list,
    then this is the list of shapes.
    y: a tensor
    y_shape: the dimensions of y as a tuple or an array of ints.
    x_init_value: (optional) a numpy array of the same shape as "x"
      representing the initial value of x. If x is a list, this should be a list
      of numpy arrays.  If this is none, the function will pick a random tensor
      as the initial value.
    delta: (optional) the amount of perturbation.
    init_targets: list of targets to run to initialize model params.
      TODO(mrry): Remove this argument.
    extra_feed_dict: dict that allows fixing specified tensor values
      during the Jacobian calculation.

  Returns:
    The maximum error in between the two Jacobians.
  """
  grad = compute_gradient(x, x_shape, y, y_shape, x_init_value, delta,
                          init_targets, extra_feed_dict=extra_feed_dict)
  if isinstance(grad, tuple):
    grad = [grad]
  error = 0
  for j_t, j_n in grad:
    if j_t.size or j_n.size:  # Handle zero size tensors correctly
      error = np.maximum(error, np.fabs(j_t - j_n).max())
  return error
```