Java Dynamic Proxy
-------------------
例如有一个HBaseClient类，但是其调用无超时，如果想对所有(或某些)方法调用加入超时逻辑，则可使用代理:  
```Java
  protected static HBaseClientInterface createClientProxy() {
    invocationHandler = new HBaseInvocationHandler(singletonClient);
    return (HBaseClientInterface) Proxy.newProxyInstance(
      HBaseClientInterface.class.getClassLoader(), new Class[] { HBaseClientInterface.class },
      invocationHandler);
  }
```
在`HBaseInvocationHandler`中重载`invoke`，用线程池实现超时(有额外开销)：  
```Java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Future<Object> result = null;
    try {
      result = this.executor.submit(new HBaseRequest(this.client, method, args));
      return result.get(requestTimeout, TimeUnit.MILLISECONDS);
    } catch (Throwable e) {
      throw new HException(e);
    }
  }
```
其中`HBaseRequest`是一个`Callable`，里边调用了真是的方法：  
```Java
  class HBaseRequest implements Callable<Object> {
    protected Object object;
    protected Method method;
    protected Object[] args;
    
    public HBaseRequest(Object object, Method method, Object[] args) {
      this.object = object;
      this.method = method;
      this.args = args;
    }

    @Override
    public Object call() throws Exception {
      return method.invoke(object, args);
    }
  }
```
类似的，通过代理可以实现其他如记录日志等操作(Spring IOP)。
