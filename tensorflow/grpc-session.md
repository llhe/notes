## 分布式执行过程分析

分布式执行与本地执行逻辑类似，这里主要介绍一下主要的区别。分布式执行主要分为两个大的步骤

* 创建并启动server
* 创建session并提交session run请求到master server

### Server的创建和启动

首先看一下创建server的代码

```py
  server = tf.train.Server(cluster,
                           job_name=FLAGS.job_name,
                           task_index=FLAGS.task_index)
```

`tf.train.Server`最终通过SWIG方式调用C++代码。

Server的创建与Session类似，也是采用工厂模式，且采用`server_factory.second->AcceptsOptions(server_def)`的方式选择合适的ServerFactory，目前开源版本仅支持grpc的实现\(TODO：RDMA实现的可行性\)。

```cpp
Status GrpcServer::Init() {
  // ...
  worker_env_.worker_cache = NewGrpcWorkerCache(channel_cache.release());

  // Finish setting up master environment.
  master_env_.ops = OpRegistry::Global();
  master_env_.worker_cache = worker_env_.worker_cache;
  master_env_.master_session_factory = [](const SessionOptions& options,
                                          const MasterEnv* env,
                                          std::vector<Device*>* remote_devs) {
    return new MasterSession(options, env, remote_devs,
                             CreateNoOpStatsPublisher);
  };

  // Finish setting up worker environment.
  worker_env_.graph_mgr = new GraphMgr(&worker_env_);
  worker_env_.compute_pool = ComputePool(sess_opts);
  worker_env_.rendezvous_mgr = new RpcRendezvousMgr(&worker_env_);

  return Status::OK();
}

Status GrpcServer::Start() {
  mutex_lock l(mu_);
  switch (state_) {
    case NEW: {
      master_thread_.reset(
          env_->StartThread(ThreadOptions(), "TF_master_service",
                            [this] { master_service_->HandleRPCsLoop(); }));
      worker_thread_.reset(
          env_->StartThread(ThreadOptions(), "TF_worker_service",
                            [this] { worker_service_->HandleRPCsLoop(); }));
      state_ = STARTED;
      LOG(INFO) << "Started server with target: " << target();
      return Status::OK();
    }
    case STARTED:
      LOG(INFO) << "Server already started (target: " << target() << ")";
      return Status::OK();
    case STOPPED:
      return errors::FailedPrecondition("Server has stopped.");
    default:
      CHECK(false);
  }
}
```

可以看到，grpc server的创建比较简单，先做一些初始化工作，然后同时启动了master和worker两个grpc服务。下面是两个服务的接口定义

```cpp
////////////////////////////////////////////////////////////////////////////////
//
// MasterService defines a TensorFlow service with which a client can
// interact to execute a distributed TensorFlow computation.
//
// A master service keeps track of multiple "master sessions". Each
// session encapsulates a computation graph and its associated state,
// and typically corresponds to a single "client session" (e.g. a
// `tensorflow::Session` instance).
//
// A session is responsible for the following:
// * assigning each node to a device (locally or remotely) using a
//   placement algorithm. This may make decisions based on collected
//   statistics from the workers in the system (e.g., memory usage,
//   bandwidth consumption, etc.)
//
// * inserting intermediate nodes and edges to support cross-device
//   and cross-process data flows and resource management.
//
// * issuing commands to workers to execute the subgraphs associated
//   with those workers.
//
// Typically, a client carries out an iterative computation
// (e.g. training) by invoking RPCs against the master in a
// client-side loop. The client first creates a client session that
// connects to a particular master (using gRPC for example). The
// master creates a corresponding master session that is hosted on
// the master and caches state between the client's invocations.
//
// After the session is established, the master returns an opaque
// handle to the client that can be used to associate the client and
// master sessions.
//
// The client may send an initial graph to the master in the
// CreateSession call, and add nodes to the graph using ExtendSession.
//
// The most frequent operation a master is "RunStep", which implements
// the `Session::Run()` API. It supports feeding in arguments,
// executing a dataflow computation, and fetching arguments.
//
// Finally, when the client no longer needs the session, it should
// close the session by invoking CloseSession, which allows the master
// to reclaim resources associated with the session. The master may
// implement a garbage collection scheme that closes sessions that
// have been inactive for some time.
//
// For example, the following pseudo-code illustrates how a client
// interacts with a master:
//
// stub = NewStub("/job:mnist/replica:0/task:0")
// {handle} = stub->CreateSession({graph_def})
// do {
//   stub->RunStep({handle, {feeds}, {fetches}})
//   // The client can evaluate a predicate locally, based on the
//   // result of `fetches`, to determine whether to terminate. For
//   // example, it might fetch the loss and evaluate whether it is less
//   // than some threshold.
// } whlie (!should_stop({fetches}));
// stub->CloseSession({handle})
//
////////////////////////////////////////////////////////////////////////////////

service MasterService {
  // Creates a session.
  rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse);

  // Extends a session.
  rpc ExtendSession(ExtendSessionRequest) returns (ExtendSessionResponse);

  // Prepares future partial run calls.
  rpc PartialRunSetup(PartialRunSetupRequest) returns (PartialRunSetupResponse);

  // Drives the graph computation.
  rpc RunStep(RunStepRequest) returns (RunStepResponse);

  // Closes a session.
  rpc CloseSession(CloseSessionRequest) returns (CloseSessionResponse);

  // List the devices usable by the master.
  rpc ListDevices(ListDevicesRequest) returns (ListDevicesResponse);

  // Close all existing sessions.
  rpc Reset(ResetRequest) returns (ResetResponse);
}
```

```cpp
////////////////////////////////////////////////////////////////////////////////
//
// WorkerService defines a TensorFlow service that executes dataflow
// graphs on a set of local devices, on behalf of a MasterService.
//
// A worker service keeps track of multiple "registered graphs". Each
// registered graph is a subgraph of a client's graph, corresponding to
// only the nodes that should execute on this worker (and any
// additional nodes necessary for inter-process communication using
// the `RecvTensor` method).
//
////////////////////////////////////////////////////////////////////////////////

service WorkerService {
  // See worker.proto for details.
  rpc GetStatus(GetStatusRequest) returns (GetStatusResponse);

  // See worker.proto for details.
  rpc RegisterGraph(RegisterGraphRequest) returns (RegisterGraphResponse);

  // See worker.proto for details.
  rpc DeregisterGraph(DeregisterGraphRequest) returns (DeregisterGraphResponse);

  // See worker.proto for details.
  rpc RunGraph(RunGraphRequest) returns (RunGraphResponse);

  // See worker.proto for details.
  rpc CleanupGraph(CleanupGraphRequest) returns (CleanupGraphResponse);

  // See worker.proto for details.
  rpc CleanupAll(CleanupAllRequest) returns (CleanupAllResponse);

  // See worker.proto for details.
  rpc RecvTensor(RecvTensorRequest) returns (RecvTensorResponse) {
    // RecvTensor Method
  }

  // See worker.proto for details.
  rpc Logging(LoggingRequest) returns (LoggingResponse);

  // See worker.proto for details.
  rpc Tracing(TracingRequest) returns (TracingResponse);
}
```

### Session创建

接下来分析一下grpc session创建过程

```cpp
// tensorflow/core/distributed_runtime/rpc/grpc_session.cc
class GrpcSessionFactory : public SessionFactory {
 public:
  Session* NewSession(const SessionOptions& options) override {
    std::unique_ptr<GrpcSession> ret;
    Status s = GrpcSession::Create(options, &ret);
    if (s.ok()) {
      return ret.release();
    } else {
      LOG(ERROR) << "Error during session construction: " << s.ToString();
      return nullptr;
    }
  }
};

Status GrpcSession::Create(const SessionOptions& options,
                           std::unique_ptr<GrpcSession>* out_session) {
  std::unique_ptr<GrpcSession> ret(new GrpcSession(options));
  SharedGrpcChannelPtr master_channel =
      NewHostPortGrpcChannel(options.target.substr(kSchemePrefixLength));
  ret->SetRemoteMaster(NewGrpcMaster(master_channel));
  *out_session = std::move(ret);
  return Status::OK();
}

void GrpcSession::SetRemoteMaster(MasterInterface* master) {
  master_.reset(master);
}
```

最终创建了一个master\_的grpc stub，用于跟master service交互。

### Session的执行

从代码逻辑上来讲，分布式执行与本地执行并没有本质区别\(placement，分割，优化，执行子图\)。通过Direct Session的执行过程可以看到，其中几个步骤需要RPC的介入：

* 注册分割后的子图\(Partition\)到对应的worker
  * WorkerService::RegisterGraph
* 执行子图
  * WorkerService::RunGraph
* 子图的输入输出
  * WorkerService::RecvTensor


下面简单分析一下具体的执行过程

grpc session run方法调用了master service的RunStep接口

```cpp
// tensorflow/core/distributed_runtime/rpc/grpc_session.cc
Status GrpcSession::Run(const RunOptions& run_options,
                        const std::vector<std::pair<string, Tensor>>& inputs,
                        const std::vector<string>& output_tensor_names,
                        const std::vector<string>& target_node_names,
                        std::vector<Tensor>* outputs,
                        RunMetadata* run_metadata) {
  // ...
  TF_RETURN_IF_ERROR(RunProto(&call_options, &req, &resp));
  // ...
}
Status GrpcSession::RunProto(CallOptions* call_options, RunStepRequest* req,
                             RunStepResponse* resp) {
  {
    mutex_lock l(mu_);
    if (handle_.empty()) {
      return errors::InvalidArgument("A session is not created yet....");
    }

    req->set_session_handle(handle_);
  }
  return master_->RunStep(call_options, req, resp);
}
```

RunStep的实现

```cpp
  // tensorflow/core/distributed_runtime/rpc/grpc_master_service.cc
  // RPC handler for running one step in a session.
  void RunStepHandler(MasterCall<RunStepRequest, RunStepResponse>* call) {
    CallOptions* call_opts = new CallOptions;
    call->SetCancelCallback([call_opts]() { call_opts->StartCancel(); });
    master_impl_->RunStep(call_opts, &call->request, &call->response,
                          [call, call_opts](const Status& status) {
                            call->ClearCancelCallback();
                            delete call_opts;
                            call->SendResponse(ToGrpcStatus(status));
                          });
    ENQUEUE_REQUEST(RunStep, true);
  }
```

```cpp
  // tensorflow/core/distributed_runtime/rpc/grpc_master_service.cc
  GrpcMasterService(MasterEnv* env, ::grpc::ServerBuilder* builder)
      : master_impl_(new Master(env, 0.0)), is_shutdown_(false) {
    builder->RegisterService(&master_service_);
    cq_ = builder->AddCompletionQueue().release();
  }
```

可以看到分布式的实现有一个通用的抽象层`MasterSession`，而grpc只是其中的一种实现方式\(非继承关系\)。

```cpp
// tensorflow/core/distributed_runtime/master.cc
void Master::RunStep(CallOptions* opts, const RunStepRequest* req,
                     RunStepResponse* resp, MyClosure done) {
  mu_.lock();
  uint64 start_time = env_->env->NowMicros();
  MasterSession* session = gtl::FindPtrOrNull(sessions_, req->session_handle());
  if (session == nullptr) {
    mu_.unlock();
    done(errors::Aborted("Session ", req->session_handle(), " is not found."));
    return;
  }
  mu_.unlock();

  SchedClosure([this, start_time, session, opts, req, resp, done]() {
    Status status = session->Run(opts, req, resp);
    uint64 done_time = env_->env->NowMicros();
    done(status);
    mutex_lock l(mu_);
    last_1000_steps_.AddValue((done_time - start_time) / 1e9);
    ++step_count_;
  });
}
```

```cpp
// tensorflow/core/distributed_runtime/master_session.h
// A session encapsulates a graph computation (resource allocation,
// placement, execution, etc.).
class MasterSession {
// ...
};
```

`MasterSession`的简要执行流程

* MasterSession::Create
  * SimpleGraphExecutionState::MakeForBaseGraph
    * InitBaseGraph
      * 执行placement
* MasterSession::Run
  * DoRunWithLocalExecution
    * BuildAndRegisterPartitions 分割并注册子图
      * calls...
        * Partition\(tensorflow\/core\/graph\/graph\_partition.h\)
        * part.worker-&gt;RegisterGraphAsync\(&c-&gt;req, &c-&gt;resp, cb\);
    * RunPartitions 执行子图
      * part.worker-&gt;RunGraphAsync
    * CleanupPartitionsAsync 清理

### Tensor的传输
与本地执行类似，每个子图的输入输出(即设备的边界)是通过容量为1的信箱(Rendezvous)来实现的。
```cpp
// A Rendezvous is an abstraction for passing a Tensor
// from a producer to a consumer, where the consumer may safely
// request the Tensor before or after it has been produced.  A
// producer never blocks when using a Rendezvous.  A consumer has the
// choice of making a blocking call or providing a callback: in either
// case, the consumer receives the Tensor as soon as it is available.
//
// A Rendezvous key encodes a single <producer, consumer> pair.  It is
// an error to call Send() or Recv*() more than once with the same
// key.
class Rendezvous : public core::RefCounted {
  // ...
  // Send() never blocks.
  virtual Status Send(const ParsedKey& key, const Args& args, const Tensor& val,
                      const bool is_dead) = 0;

  // Callback provided by a tensor consumer waiting on the rendezvous.
  // It will be invoked when the tensor is available, or when a non-OK
  // status arises in the production of that tensor.  It also gets
  // two Rendezvous::Args, one provided by the sender, the other by the
  // receiver, which may be needed when a non-CPU device is in use
  // by either side.
  typedef std::function<void(const Status&, const Args&, const Args&,
                             const Tensor&, const bool)>
      DoneCallback;

  virtual void RecvAsync(const ParsedKey& key, const Args& args,
                         DoneCallback done) = 0;

  // Synchronous wrapper for RecvAsync.
  Status Recv(const ParsedKey& key, const Args& args, Tensor* val,
              bool* is_dead, int64 timeout_ms);
  Status Recv(const ParsedKey& key, const Args& args, Tensor* val,
              bool* is_dead);

};
```
分布式环境下的`Rendezvous`实现
```cpp
  // tensorflow/core/distributed_runtime/rpc/rpc_rendezvous_mgr.cc
class RpcRecvTensorCall : public BaseRecvTensorCall {
  // ...
  void Start(std::function<void()> recv_done) override {
    StartRTCall(std::move(recv_done));
  }

  // Start the main RecvTensor call, checking for an async abort.
  void StartRTCall(std::function<void()> recv_done) {
    resp_.InitAlloc(dst_device_, alloc_attrs_);
    using namespace std::placeholders;
    StatusCallback cb = std::bind(
        [this](std::function<void()> recv_done,
               // Begin unbound arguments.
               const Status& s) {
          if (!s.ok()) {
            mutex_lock l(mu_);
            status_.Update(s);
          }
          recv_done();
        },
        std::move(recv_done), _1);
    wi_->RecvTensorAsync(&opts_, &req_, &resp_, std::move(cb));
  }
  // ...
};

void RpcRemoteRendezvous::RecvFromRemoteAsync(
    const Rendezvous::ParsedKey& parsed, const Rendezvous::Args& recv_args,
    DoneCallback done) {
  // ...
  // Prepare a RecvTensor call that can handle being aborted.
  RpcRecvTensorCall* call = get_call_freelist()->New();
  // ...
  call->Start([this, call]() {
    // Removes "call" from active_. Prevent StartAbort().
    DeregisterCall(call);
    // If StartAbort was called prior to DeregisterCall, then the
    // current status should be bad.
    Status s = call->status();
    call->done()(s, Args(), call->recv_args(), call->tensor(), call->is_dead());
    cache_->ReleaseWorker(call->src_worker_, call->wi_);
    call->wi_ = nullptr;
    get_call_freelist()->Release(call);
    Unref();
  });
}
```
从代码中可以看到，`RecvFromRemoteAsync`实际上调用了`RecvTensorAsync` RPC调用来获取输入。
