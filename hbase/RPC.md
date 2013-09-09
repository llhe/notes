1. 序列化
默认的序列化采用Hadoop的Writable接口，即手工进行序列化

2. 涉及到的类和接口
  * org.apache.hadoop.hbase.client.HTableInterface：用户的接口
  * org.apache.hadoop.hbase.client.HTable
  * org.apache.hadoop.hbase.client.HConnection：定义了cluster的接口，其中`getHRegionConnection`返回`HRegionInterface`
  * org.apache.hadoop.hbase.client.HConnectionImplementation
  * org.apache.hadoop.hbase.ipc.HRegionInterface: 真正的region的rpc接口

3. 调用关系
HTableInterface/HTable定义实现了table的CRUD操作，是最终client的使用接口。
其含有成员HConnection connection初始化的时候
```Java
  private void initialize(Configuration conf)
      throws IOException {
    if (conf == null) {
      this.connection = null;
      return;
    }
    this.connection = HConnectionManager.getConnection(conf);
```
而`HConnectionManager.getConnection(conf)`返回的是一个`HConnectionImplementation:HConnection`的实例：
```Java
  public static HConnection getConnection(Configuration conf)
  throws ZooKeeperConnectionException {
    HConnectionKey connectionKey = new HConnectionKey(conf);
    synchronized (HBASE_INSTANCES) {
      HConnectionImplementation connection = HBASE_INSTANCES.get(connectionKey);
      if (connection == null) {
        connection = new HConnectionImplementation(conf, true, null);
        HBASE_INSTANCES.put(connectionKey, connection);
```
对于某个调用，如increment：
```Java
  @Override
  public Result increment(final Increment increment) throws IOException {
    if (!increment.hasFamilies()) {
      throw new IOException(
          "Invalid arguments to increment, no columns specified");
    }
    return new ServerCallable<Result>(connection, tableName, increment.getRow(), operationTimeout) {
          public Result call() throws IOException {
            return server.increment(
                location.getRegionInfo().getRegionName(), increment);
          }
        }.withRetries();
  }
```
创建了`ServerCallable<T>::Callable<T>`，并实现了其`call`方法，`call`方法中调用`server.increment`，并调用其`withRetries`方法。`ServerCallable::withRetries`中调用了`call`方法：
```Java
  public T withoutRetries()
  throws IOException, RuntimeException {
    try {
      beforeCall();
      connect(false); // 会对server变量进行赋值
      return call();
    }
```
接下来需要看`server.increment`的调用流程。其中`HRegionInterface server`在`connect`中被赋值：
```Java
  /**
   * Connect to the server hosting region with row from tablename.
   * @param reload Set this to true if connection should re-find the region
   * @throws IOException e
   */
  public void connect(final boolean reload) throws IOException {
    long startTime = System.currentTimeMillis();
    // locate region will connect zk and has recursive looks up process
    this.location = connection.getRegionLocation(tableName, row, reload);
    long callTime = System.currentTimeMillis() - startTime;
    if (callTime - startTime > 100) {
      LOG.warn("Slow locate region, reload=" + reload + ", tableName=" + Bytes.toString(tableName)
          + ", time consume=" + callTime);
    }
    startTime = System.currentTimeMillis();
    // get server proxy may need create socket connection to server
    this.server = connection.getHRegionConnection(location.getHostname(),
      location.getPort());
    callTime = System.currentTimeMillis() - startTime;
    if (callTime - startTime > 100) {
      LOG.warn("Slow get server proxy, reload=" + reload + ", tableName=" + Bytes.toString(tableName)
          + ", time consume=" + callTime);
    }
  }
```
前面已经提到，这里的connection来自HTable中的`HConnectionManager.getConnection(conf)`调用，即`HConnectionImplementation`。`HConnectionImplementation.getHRegionConnection`中：
```Java
    HRegionInterface getHRegionConnection(final String hostname, final int port,
        final InetSocketAddress isa, final boolean master)
    throws IOException {
      if (master) getMaster();
      HRegionInterface server;
      String rsName = null;
      if (isa != null) {
        rsName = Addressing.createHostAndPortStr(isa.getHostName(),
            isa.getPort());
      } else {
        rsName = Addressing.createHostAndPortStr(hostname, port);
      }
      ensureZookeeperTrackers();
      // See if we already have a connection (common case)
      server = this.servers.get(rsName);
      if (server == null) {
        // create a unique lock for this RS (if necessary)
        this.connectionLock.putIfAbsent(rsName, rsName);
        // get the RS lock
        synchronized (this.connectionLock.get(rsName)) {
          // do one more lookup in case we were stalled above
          server = this.servers.get(rsName);
          if (server == null) {
            try {
              if (clusterId.hasId()) {
                conf.set(HConstants.CLUSTER_ID, clusterId.getId());
              }
              // Only create isa when we need to.
              InetSocketAddress address = isa != null? isa:
                new InetSocketAddress(hostname, port);
              // definitely a cache miss. establish an RPC for this RS
              long startTime = System.currentTimeMillis();
              server = (HRegionInterface) HBaseRPC.waitForProxy(
                  serverInterfaceClass, HRegionInterface.VERSION,
                  address, this.conf,
                  this.maxRPCAttempts, this.rpcTimeout, this.rpcTimeout);
              LOG.info("create proxy for region server:" + address + ", rpcTimeout="
                  + this.rpcTimeout + ", time consume=" + (System.currentTimeMillis() - startTime));
              this.servers.put(Addressing.createHostAndPortStr(
                  address.getHostName(), address.getPort()), server);
            } catch (RemoteException e) {
              LOG.warn("RemoteException connecting to RS", e);
              // Throw what the RemoteException was carrying.
              throw e.unwrapRemoteException();
            }
          }
        }
      }
      return server;
    }
```
代码中可见`HRegionInterface`是有缓存的，且实例的获取是加锁的，注意加锁代码，视通过`ConcurrentMap`来做锁管理器的。接下来看一下`HRegionInterface`是如何创建的:
```Java
              server = (HRegionInterface) HBaseRPC.waitForProxy(
                  serverInterfaceClass, HRegionInterface.VERSION,
                  address, this.conf,
                  this.maxRPCAttempts, this.rpcTimeout, this.rpcTimeout);
```
其中`serverInterfaceClass`为：
```Java
        // serverClassName 默认为HRegionInterface.class.getName()
        this.serverInterfaceClass =
          (Class<? extends HRegionInterface>) Class.forName(serverClassName);
```
`HBaseRPC.waitForProxy`的实现:
```Java
  public static VersionedProtocol waitForProxy(Class protocol,
                                               long clientVersion,
                                               InetSocketAddress addr,
                                               Configuration conf,
                                               int maxAttempts,
                                               int rpcTimeout,
                                               long timeout
                                               ) throws IOException {
    // HBase does limited number of reconnects which is different from hadoop.
    long startTime = System.currentTimeMillis();
    IOException ioe;
    int reconnectAttempts = 0;
    while (true) {
      try {
        return getProxy(protocol, clientVersion, addr, conf, rpcTimeout);
```
`getProxy`依次调用到:
```Java
    VersionedProtocol proxy =
        getProtocolEngine(protocol,conf)
            .getProxy(protocol, clientVersion, addr, ticket, conf, factory, Math.min(rpcTimeout, HBaseRPC.getRpcTimeout()));
    long serverVersion = proxy.getProtocolVersion(protocol.getName(),
                                                  clientVersion);
```
其中`getProtocolEngine(protocol,conf)`代码如下:
```Java
  // return the RpcEngine configured to handle a protocol
  private static synchronized RpcEngine getProtocolEngine(Class protocol,
                                                          Configuration conf) {
    RpcEngine engine = PROTOCOL_ENGINES.get(protocol);
    if (engine == null) {
      // check for a configured default engine
      Class<?> defaultEngine =
          conf.getClass(RPC_ENGINE_PROP, WritableRpcEngine.class);

      // check for a per interface override
      Class<?> impl = conf.getClass(RPC_ENGINE_PROP+"."+protocol.getName(),
                                    defaultEngine);
      LOG.debug("Using "+impl.getName()+" for "+protocol.getName());
      engine = (RpcEngine) ReflectionUtils.newInstance(impl, conf);
      if (protocol.isInterface())
        PROXY_ENGINES.put(Proxy.getProxyClass(protocol.getClassLoader(),
                                              protocol),
                          engine);
      PROTOCOL_ENGINES.put(protocol, engine);
    }
    return engine;
  }
```
可见，默认情况(RPC_ENGINE_PROP = "hbase.rpc.engine"未配置)，engine实际是一个WritableRpcEngine的实例(`engine = (RpcEngine) ReflectionUtils.newInstance(impl, conf);`)。
