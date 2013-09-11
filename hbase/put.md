HBase Put流程分析
===================
Client流程
-----------------
HTable.java
Put 调用doPut

doPut(List):
先写入缓存中(简单的list，非线程安全)，autoFlush或者每DOPUT_WB_CHECK次或者buffer大小超过阈值调用flushCommits()

flushCommits:
调用this.connection.processBatch(writeBuffer, tableName, pool, results);写入

processBatch调用processBatchCallback

processBatchCallback:
对buffer中的请求按照RS(根据以前缓存的信息)进行分发得到Map<HRegionLocation, MultiAction<R>> actionsByServer。

然后利用一个线程池并发进行RPC调用
```Java
        for (Entry<HRegionLocation, MultiAction<R>> e: actionsByServer.entrySet()) {
          futures.put(e.getKey(), pool.submit(createCallable(e.getKey(), e.getValue(), tableName)));
        }
```
收集调用结果
```Java
        // step 3: collect the failures and successes and prepare for retry

        for (Entry<HRegionLocation, Future<MultiResponse>> responsePerServer
             : futures.entrySet()) {
          HRegionLocation loc = responsePerServer.getKey();

          try {
            Future<MultiResponse> future = responsePerServer.getValue();
            MultiResponse resp = future.get();

            if (resp == null) {
              // Entire server failed
              LOG.debug("Failed all for server: " + loc.getHostnamePort() +
                ", removing from cache");
              continue;
            }

            for (Entry<byte[], List<Pair<Integer,Object>>> e : resp.getResults().entrySet()) {
              byte[] regionName = e.getKey();
              List<Pair<Integer, Object>> regionResults = e.getValue();
              for (Pair<Integer, Object> regionResult : regionResults) {
                if (regionResult == null) {
                  // if the first/only record is 'null' the entire region failed.
                  LOG.debug("Failures for region: " +
                      Bytes.toStringBinary(regionName) +
                      ", removing from cache");
                } else {
                  // Result might be an Exception, including DNRIOE
                  results[regionResult.getFirst()] = regionResult.getSecond();
                  if (callback != null && !(regionResult.getSecond() instanceof Throwable)) {
                    callback.update(e.getKey(),
                        list.get(regionResult.getFirst()).getRow(),
                        (R)regionResult.getSecond());
                  }
                }
              }
            }
          } catch (ExecutionException e) {
            LOG.warn("Failed all from " + loc, e);
          }
        }
```

根据调用结果出错信息判断是否更新region信息(如果不是DoNotRetryIOException)并重试
```Java
        for (int i = 0; i < results.length; i++) {
          // if null (fail) or instanceof Throwable && not instanceof DNRIOE
          // then retry that row. else dont.
          if (results[i] == null ||
              (results[i] instanceof Throwable &&
                  !(results[i] instanceof DoNotRetryIOException))) {

            retry = true;
            actionCount++;
            Row row = list.get(i);
            workingList.add(row);
            deleteCachedLocation(tableName, row.getRow());
          } else {
            if (results[i] != null && results[i] instanceof Throwable) {
              actionCount++;
            }
            // add null to workingList, so the order remains consistent with the original list argument.
            workingList.add(null);
          }
        }
```

createCallable创建进行进行RPC调用的实体，最终调用的RPC是`multi`：
```Java
       public MultiResponse call() throws IOException {
         ServerCallable<MultiResponse> callable =
           new ServerCallable<MultiResponse>(connection, tableName, null) {
             public MultiResponse call() throws IOException {
               return server.multi(multi);
             }
             @Override
             public void connect(boolean reload) throws IOException {
               server = connection.getHRegionConnection(loc.getHostname(), loc.getPort());
             }
           };
         return callable.withoutRetries();
       }
```

RS流程
----------------
HBase Region Server是自己实现的RPC(只支持Java，另外通过Thrift支持其他语言)，Avro序列化+TCP server+java反射/代理等(具体实现待分析)，HRegionInterface定义了CS交互的接口，由client和server作为协议共享，HRegionServer实现了此接口：
```Java
public class HRegionServer implements HRegionInterface, HBaseRPCErrorHandler,
    Runnable, RegionServerServices {
      // ...
    }
```
```Java
    this.rpcServer = HBaseRPC.getServer(this,
      new Class<?>[]{HRegionInterface.class, HBaseRPCErrorHandler.class,
        OnlineRegions.class},
        initialIsa.getHostName(), // BindAddress is IP we got for this server.
        initialIsa.getPort(),
        conf.getInt("hbase.regionserver.handler.count", 10),
        conf.getInt("hbase.regionserver.metahandler.count", 10),
        conf.getBoolean("hbase.rpc.verbose", false),
        conf, HConstants.QOS_THRESHOLD);
```
接下来是`multi`的处理:
```Java
  public <R> MultiResponse multi(MultiAction<R> multi) throws IOException {
  }
```
multi根据不同region，不同的操作调用具体的方法，此处调用put：
```Java
          if (action instanceof Delete || action instanceof Put) {
            mutations.add(a); 
          }
```

一个HRegion管理一个Region，包含一段row的所有column，其中每个CF对应一个HStore(负责MemStore和StoreFile的管理，WAL和lock在HRegion层次处理)。

最后到达`HRegion.java:doMiniBatchMutation`：


问题：
是否flush的时候写入是阻塞的？即没有leveldb对应的immutable memstore？
