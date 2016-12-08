## 本地执行过程分析

一个graph的真正执行是通过session run来完成的，本地执行称为direct session。这里分析direct session执行过程，下面是python层的调用接口

```py
sess = tf.Session(config=sess_config)
sess.run([train_step])
```

### SWIG接口封装

首先看一下tf.Session和sess.run方法如何import而来：

```py
# tensorflow/python/__init__.py
from tensorflow.python.client.client_lib import *
```

```py
# tensorflow/python/client/client_lib.py
from tensorflow.python.client.session import Session
```

#### tf.Session

`tf.Session`中调用`tf_session.TF_NewDeprecatedSession`创建一个session实例

```py
# tensorflow/python/client/session.py
def __init__(self, target='', graph=None, config=None):
    # ...
    try:
      with errors.raise_exception_on_not_ok_status() as status:
        self._session = tf_session.TF_NewDeprecatedSession(opts, status)
    finally:
      tf_session.TF_DeleteSessionOptions(opts)
```

而`tf_session.TF_NewDeprecatedSession`为SWIG封装的C接口

```py
# tensorflow/python/client/tf_session.i
%unignore TF_NewDeprecatedSession;
```

#### sess.run

跟踪session.py的`Session`类的`run`方法，最终在跟踪到`_run_fn`方法，此方法调用了`tf_session.TF_Run`：

```py
  # tensorflow/python/client/session.py
    def _run_fn(session, feed_dict, fetch_list, target_list, options,
                run_metadata):
      # Ensure any changes to the graph are reflected in the runtime.
      self._extend_graph()
      with errors.raise_exception_on_not_ok_status() as status:
        return tf_session.TF_Run(session, options,
                                 feed_dict, fetch_list, target_list,
                                 status, run_metadata)
```

而`tf_session.TF_Run`是通过SWIG封装的C代码：

```c
// tensorflow/python/client/tf_session.i
// Include the wrapper for TF_Run from tf_session_helper.h.

// The %exception block above releases the Python GIL for the length
// of each wrapped method. We disable this behavior for TF_Run
// because it uses the Python allocator.
%noexception tensorflow::TF_Run_wrapper;
%rename(TF_Run) tensorflow::TF_Run_wrapper;
```

TF\_Run实际通过TF\_Run\_wrapper重命名而来：

```cpp
// tensorflow/python/client/tf_session_helper.cc
void TF_Run_wrapper(TF_DeprecatedSession* session, const TF_Buffer* run_options,
                    PyObject* feed_dict, const NameVector& output_names,
                    const NameVector& target_nodes, TF_Status* out_status,
                    PyObjectVector* out_values, TF_Buffer* run_outputs) {
  TF_Run_wrapper_helper(session, nullptr, run_options, feed_dict, output_names,
                        target_nodes, out_status, out_values, run_outputs);
}

void TF_Run_wrapper_helper(TF_DeprecatedSession* session, const char* handle,
                           const TF_Buffer* run_options, PyObject* feed_dict,
                           const NameVector& output_names,
                           const NameVector& target_nodes,
                           TF_Status* out_status, PyObjectVector* out_values,
                           TF_Buffer* run_outputs) {
  // ...
  if (handle == nullptr) {
    TF_Run(session, run_options, input_names.data(), inputs_unsafe.data(),
           input_names.size(), const_cast<const char**>(output_names.data()),
           outputs.data(), output_names.size(),
           const_cast<const char**>(target_nodes.data()), target_nodes.size(),
           run_outputs, out_status);
  } else {
    TF_PRun(session, handle, input_names.data(), inputs_unsafe.data(),
            input_names.size(), const_cast<const char**>(output_names.data()),
            outputs.data(), output_names.size(),
            const_cast<const char**>(target_nodes.data()), target_nodes.size(),
            out_status);
  }
  // ...
}
```

最终调用到`tensorflow/c/c_api.h`中的TF\_Run

```cpp
    // tensorflow/c/c_api.cc
    result = session->Run(run_options_proto, input_pairs, output_tensor_names,
                          target_oper_names, &outputs, &run_metadata_proto);
```

最后，调用了C++对应`Session`实现的`Run`方法。本地模式执行，即调用`DirectSession::Run`。

### Session的创建

前面提到，Session通过C接口的`TF_DeprecatedSession`来创建，其定义如下

```cpp
// tensorflow/c/c_api.cc
struct TF_DeprecatedSession {
  Session* session;
};

TF_DeprecatedSession* TF_NewDeprecatedSession(const TF_SessionOptions* opt,
                                              TF_Status* status) {
  Session* session;
  status->status = NewSession(opt->options, &session);
  if (status->status.ok()) {
    return new TF_DeprecatedSession({session});
  } else {
    DCHECK_EQ(nullptr, session);
    return NULL;
  }
}
```

```cpp
// tensorflow/core/common_runtime/session.cc
Session* NewSession(const SessionOptions& options) {
  SessionFactory* factory;
  Status s = SessionFactory::GetFactory(options, &factory);
  if (!s.ok()) {
    LOG(ERROR) << s;
    return nullptr;
  }
  return factory->NewSession(options);
}
```

可以看到`Session`通过`SessionFactory`来创建。而`GetFactory`过程并非通过注册的名字\(目前实现有`DIRECT_SESSION`和`GRPC_SESSION`两种\)来得到对应的Factory，而是遍历所有注册过的Factory，根据Session options来决定哪个Factory合适。

```cpp
Status SessionFactory::GetFactory(const SessionOptions& options,
                                  SessionFactory** out_factory) {
  mutex_lock l(*get_session_factory_lock());  // could use reader lock

  std::vector<std::pair<string, SessionFactory*>> candidate_factories;
  for (const auto& session_factory : *session_factories()) {
    if (session_factory.second->AcceptsOptions(options)) {
      VLOG(2) << "SessionFactory type " << session_factory.first
              << " accepts target: " << options.target;
      candidate_factories.push_back(session_factory);
    } else {
      VLOG(2) << "SessionFactory type " << session_factory.first
              << " does not accept target: " << options.target;
    }
  }
  // ...
}
```

下面是`DirectSessionFactory`的实现：

```cpp
// tensorflow/core/common_runtime/direct_session.cc
class DirectSessionFactory : public SessionFactory {
 public:
  DirectSessionFactory() {}

  bool AcceptsOptions(const SessionOptions& options) override {
    return options.target.empty();
  }

  Session* NewSession(const SessionOptions& options) override {
    // Must do this before the CPU allocator is created.
    if (options.config.graph_options().build_cost_model() > 0) {
      EnableCPUAllocatorFullStats(true);
    }
    std::vector<Device*> devices;
    Status s = DeviceFactory::AddDevices(
        options, "/job:localhost/replica:0/task:0", &devices);
    if (!s.ok()) {
      LOG(ERROR) << s;
      return nullptr;
    }

    DirectSession* session =
        new DirectSession(options, new DeviceMgr(devices), this);
    {
      mutex_lock l(sessions_lock_);
      sessions_.push_back(session);
    }
    return session;
  }
  // ...
}
```

### DirectSession Run逻辑

首先看一下`DirectSession`的定义
```cpp
// tensorflow/core/common_runtime/direct_session.h

class DirectSession : public Session {
  // ...
  // We create one executor and its dependent library runtime for
  // every partition.
  struct PerPartitionExecutorsAndLib {
    Graph* graph = nullptr;
    std::unique_ptr<FunctionLibraryRuntime> flib;
    std::unique_ptr<Executor> executor;
  };

  // An ExecutorsAndKeys is created for a given set of feeds/fetches.
  // 'step_count' is the number of times this graph is executed.
  // 'graph' is the entire graph being executed. 'name_to_node'
  // maps node name to node. We keep 'graph' and 'name_to_node' only in
  // the case of partial runs. Each item in 'items' is the executor for
  // a partition of the graph bundled with its dependent library runtime.
  // 'input_keys' are the rendezvous keys for the feeds and 'output_keys'
  // are rendezvous keys for the fetches.
  // 'flib_def' is the function library used by graphs in 'items'.
  // TODO(phawkins): currently partitions always share the same function
  // library. Consider giving each partition its own function library to enable
  // per-partition rewrites.
  struct ExecutorsAndKeys {
    int64 step_count = 0;
    std::unique_ptr<Graph> graph;
    NameNodeMap name_to_node;
    std::unique_ptr<FunctionLibraryDefinition> flib_def;
    std::vector<PerPartitionExecutorsAndLib> items;
    std::unordered_map<string, string> input_keys;
    std::unordered_map<string, string> output_keys;
  };

  // For each live partial execution, the session maintains a RunState.
  // 'status' is the current status of this partial execution. 'executor_done'
  // is "notified" when all executors are done. 'pending_inputs' are the set
  // of pending feeds and 'pending_outputs' are the set of pending fetches.
  struct RunState {
    mutex mu_;
    Status status GUARDED_BY(mu_);
    IntraProcessRendezvous* rendez = nullptr;
    std::unique_ptr<StepStatsCollector> collector;
    Notification executors_done;
    std::unordered_set<string> pending_inputs;
    std::unordered_set<string> pending_outputs;
    TensorStore tensor_store;
    ResourceMgr step_resource_manager;

    RunState(const std::vector<string>& input_names,
             const std::vector<string>& output_names);

    ~RunState();
  };

  // 设备信息
  const std::unique_ptr<const DeviceMgr> device_mgr_;
  std::vector<Device*> devices_;  // not owned
  DeviceSet device_set_;

  // 线程池
  std::vector<thread::ThreadPool*> thread_pools_;

  // Holds mappings from signature to the executors that process
  // it. The reason for a level of indirection around mapped_type is
  // to guarantee address stability.
  std::unordered_map<string, std::unique_ptr<ExecutorsAndKeys>> executors_
      GUARDED_BY(executor_lock_);

  // 执行状态，可以同时有多个并发执行
  // Holds mappings from handle to partial run state.
  std::unordered_map<string, std::unique_ptr<RunState>> partial_runs_
      GUARDED_BY(executor_lock_);
}
```
DriectSession中定义了`ExecutorsAndKeys`结构，用于执行一个Graph\(执行时会根据设备分配情况分割成不同的子图，分别执行\)。

接下来看一下`DirectSession::Run`的执行过程：

```cpp
// tensorflow/core/common_runtime/direct_session.cc
Status DirectSession::Run(const RunOptions& run_options,
                          const NamedTensorList& inputs,
                          const std::vector<string>& output_names,
                          const std::vector<string>& target_nodes,
                          std::vector<Tensor>* outputs,
                          RunMetadata* run_metadata) {
  // ...
  // 获取线程池
  thread::ThreadPool* pool = thread_pools_[run_options.inter_op_thread_pool()];
  // ...
  ExecutorsAndKeys* executors_and_keys;
  TF_RETURN_IF_ERROR(
      GetOrCreateExecutors(pool, input_tensor_names, output_names, target_nodes,
                           &executors_and_keys, &run_state_args));
  // ...
  // 会在executor.cc中调用，提交计算任务到线程池中执行
  args.runner = [this, pool](Executor::Args::Closure c) {
    SchedClosure(pool, std::move(c));
  };
  // ...
  for (const auto& item : executors_and_keys->items) {
    // 执行每个子图(partition)
    item.executor->RunAsync(args, barrier->Get());
  }
  // ...
}

Status DirectSession::GetOrCreateExecutors(
    thread::ThreadPool* pool, gtl::ArraySlice<string> inputs,
    gtl::ArraySlice<string> outputs, gtl::ArraySlice<string> target_nodes,
    ExecutorsAndKeys** executors_and_keys, RunStateArgs* run_state_args) {
  // ...
  // 创建graph，并根据placement算法分割成不同的子图(partition)
  std::unordered_map<string, std::unique_ptr<Graph>> graphs;
  TF_RETURN_IF_ERROR(
      CreateGraphs(options, &graphs, &ek->flib_def, run_state_args));
  // ...

  const auto& optimizer_opts =
      options_.config.graph_options().optimizer_options();
  GraphOptimizer optimizer(optimizer_opts);
  for (auto iter = graphs.begin(); iter != graphs.end(); ++iter) {
    // ...
    // 对每个partition执行优化算法
    optimizer.Optimize(lib, options_.env, device, &partition_graph);
    // 初始化设备，kernel等...
    // ...
    // 为每个子图设置executor
    // NewLocalExecutor takes ownership of partition_graph.
    item->graph = partition_graph;
    item->executor = nullptr;
    Executor* executor;
    TF_RETURN_IF_ERROR(
        NewLocalExecutor(params, iter->second.release(), &executor));
    item->executor.reset(executor);
  }
}
```

从Run方法的代码中可以看到，代码逻辑如下：

* 获取线程池
* 生成执行子图
  * 根据client graph生成base graph
  * 执行placement算法将计算节点分配到不同的设备
  * 根据分配结果对图进行分割，得到一系列子图\(不同location\)
* 对子图进行优化
* 执行各个子图

### 生成执行子图

生成Partition的主要步骤：

* 根据client graph生成base graph
* 调用SimplePlacer进行Device分配
* 调用Partition方法，分割成不同子图

  ```cpp
  // tensorflow/core/common_runtime/direct_session.cc
  Status DirectSession::CreateGraphs(
    const BuildGraphOptions& subgraph_options,
    std::unordered_map<string, std::unique_ptr<Graph>>* outputs,
    std::unique_ptr<FunctionLibraryDefinition>* flib_def,
    RunStateArgs* run_state_args) {
  mutex_lock l(graph_def_lock_);
  std::unique_ptr<SimpleClientGraph> client_graph;

  std::unique_ptr<SimpleGraphExecutionState> temp_exec_state_holder;
  SimpleGraphExecutionState* execution_state = nullptr;
  // 创建base graph
  if (options_.config.graph_options().place_pruned_graph()) {
    // Because we are placing pruned graphs, we need to create a
    // new SimpleGraphExecutionState for every new unseen graph,
    // and then place it.
    SimpleGraphExecutionStateOptions prune_options;
    prune_options.device_set = &device_set_;
    prune_options.session_options = &options_;
    prune_options.stateful_placements = stateful_placements_;
    TF_RETURN_IF_ERROR(SimpleGraphExecutionState::MakeForPrunedGraph(
        execution_state_->original_graph_def().library(), prune_options,
        execution_state_->original_graph_def(), subgraph_options,
        &temp_exec_state_holder, &client_graph));
    execution_state = temp_exec_state_holder.get();
  } else {
    execution_state = execution_state_.get();
    TF_RETURN_IF_ERROR(
        execution_state->BuildGraph(subgraph_options, &client_graph));
  }

  // 执行placement算法
  auto current_stateful_placements = execution_state->GetStatefulPlacements();
  // ...
  // 根据placement结果分割成子图
  std::unordered_map<string, GraphDef> partitions;
  TF_RETURN_IF_ERROR(Partition(popts, &client_graph->graph, &partitions));
```

```cpp
// tensorflow/core/common_runtime/simple_graph_execution_state.h

// SimpleGraphExecutionState is responsible for generating an
// executable SimpleClientGraph from the original GraphDef that specifies
// the complete graph and from BuildGraphOptions which specifies
// input/output nodes.
//
// An executable Graph differs from a GraphDef by being Placed,
// meaning that each Node is assigned to a single Device in the
// available set.
//
// When SimpleGraphExecutionState is first constructed it instantiates
// a full Graph from the provided GraphDef, and places it, using only
// the static device assignments from the GraphDef.  Nodes without are
// currently placed in a very naive way.  Since stateful Nodes cannot
// be moved after initial placement, it is important that stateful
// Nodes get sensible initial device assignments in the graph
// definition.
//
// Subsequently, SimpleGraphExecutionState generates a SimpleClientGraph on
// demand, which is a sub-graph of the latest placement of the full
// Graph.  MasterSession uses such a SimpleClientGraph to execute one or
// more similar client requests.
//
// SimpleGraphExecutionState is thread-safe.
```

```cpp
// tensorflow/core/common_runtime/simple_graph_execution_state.cc
/* static */ Status SimpleGraphExecutionState::MakeForBaseGraph(
    GraphDef* graph_def, const SimpleGraphExecutionStateOptions& options,
    std::unique_ptr<SimpleGraphExecutionState>* out_state) {
  std::unique_ptr<SimpleGraphExecutionState> ret(
      new SimpleGraphExecutionState(graph_def, options));

  TF_RETURN_IF_ERROR(AddDefaultAttrsToGraphDef(&ret->original_graph_def_,
                                               *ret->flib_def_.get(), 0));
  // TODO(mrry): Refactor InitBaseGraph() so that we don't have to
  // pass an empty BuildGraphOptions (that isn't going to be used when
  // place_pruned_graph is false).
  if (!ret->session_options_->config.graph_options().place_pruned_graph()) {
    TF_RETURN_IF_ERROR(ret->InitBaseGraph(BuildGraphOptions()));
  }
  *out_state = std::move(ret);
  return Status::OK();
}

Status SimpleGraphExecutionState::InitBaseGraph(...) {
  // ...
  SimplePlacer placer(new_graph.get(), device_set_, session_options_);
  // TODO(mrry): Consider making the SimplePlacer cancelable.
  TF_RETURN_IF_ERROR(placer.Run());
  // 最终调用的SimplePlacer
```

```cpp
// tensorflow/core/graph/graph_partition.h
// Partition "input" graph into a set of graphs, one per location.
// The location for node n is derived by calling opts.node_to_loc(n).
// New nodes added by Partition use "opts.new_name(old_name)" to
// generate node names.
//
// Stores the partitions in *partitions.
Status Partition(const PartitionOptions& opts, Graph* input,
                 std::unordered_map<string, GraphDef>* partitions);
```

#### Placement算法

目前代码仅支持`SimplePlacer`，主要用到了[并查集](https://en.wikipedia.org/wiki/Disjoint-set_data_structure)。

```cpp
// tensorflow/core/common_runtime/simple_placer.h

// A placement algorithm that assigns the nodes of the given Graph to
// devices the given DeviceSet, respecting the following constraints:
//
// 1. Existing device assignments remain unchanged.
// 2. Requested (partial or complete) device specifications given by device name
//    for each node are granted.
// 3. Nodes connected by edges of a reference type are colocated on
//    the same device.
// 4. Given nodes "A" and "B", if node "B" has a colocation group
//    "@loc:A", nodes "A" and "B" will be colocated on the same device.

class SimplePlacer {
 public:
  // Assigns each node in this SimplePlacer's graph to a device in its
  // set of devices.
  //
  // This method is not thread-safe.
  // Run() may be invoked at most once.
  Status Run();

 private:
  // Returns true if the device type of 'candidate_device_name' is
  // found in 'devices'.
  bool CanAssignToDevice(const string& candidate_device_name,
                         const std::vector<Device*> devices) const;

  // Assigns 'node's devices to 'assigned_device', and logs the
  // placement if the SessionOptions entry in 'options_' requests it.
  void AssignAndLog(const string& assigned_device, Node* node) const;
}
```

SimplePlacer的Run方法逻辑：

```cpp
// ./tensorflow/core/common_runtime/simple_placer.cc
Status SimplePlacer::Run() {
  // 1. First add all of the nodes. Note that steps (1) and (2)
  // requires two passes over the nodes because the graph (and hence
  // the constraints) may not be acyclic.

  // 2. Enumerate the constraint edges, and use them to update the disjoint
  // node set.

  // 3. For each node, assign a device based on the constraints in the
  // disjoint node set.

  // 4. Perform a second pass assignment for those nodes explicitly
  // skipped during the first pass.
}
```

### 子图的优化

图的优化目前比较简单，主要有下面几种：
* 合并冗余List Array转换代码
  * Rewrites _ListToArray and _ArrayToList to a set of Identity nodes
* 删除不可达结点
* 删除Identy节点(noop)
* 常量折叠
* FixupSourceAndSinkEdges(?)
  * Connect all nodes with no incoming edges to source.
  * Connect all nodes with no outgoing edges to sink.
* 公共子表达式消除
* 函数调用内联
  * TODO

```cpp
void GraphOptimizer::Optimize(FunctionLibraryRuntime* runtime, Env* env,
                              Device* device, Graph** graph) {
  Graph* g = *graph;
  DumpGraph("Initial", g);

  bool changed = true;
  const int kMaxRounds = 10;
  for (int rounds = 0; rounds < kMaxRounds; ++rounds) {
    changed = false;
    if (RemoveListArrayConverter(g)) {
      DumpGraph("RemoveListArrayConverter", g);
      changed = true;
    }
    if (opts_.do_function_inlining() && RemoveDeadNodes(g)) {
      DumpGraph("RemoveDeadNodes", g);
      changed = true;
    }
    if (opts_.do_function_inlining() && RemoveIdentityNodes(g)) {
      DumpGraph("RemoveIdentityNodes", g);
      changed = true;
    }

    if (opts_.do_constant_folding()) {
      ConstantFoldingOptions cf_opts;
      if (DoConstantFolding(cf_opts, runtime, env, device, g)) {
        RemoveDeadNodes(g);
        DumpGraph("ConstFolding", g);
        changed = true;
      }
    }

    if (opts_.do_function_inlining() && FixupSourceAndSinkEdges(g)) {
      DumpGraph("FixupSourceAndSinkEdges", g);
      changed = true;
    }
    if (opts_.do_common_subexpression_elimination() &&
        OptimizeCSE(g, nullptr)) {
      DumpGraph("OptimizeCSE", g);
      changed = true;
    }
    if (opts_.do_function_inlining() && ExpandInlineFunctions(runtime, g)) {
      DumpGraph("ExpandInlineFunctions", g);
      changed = true;
    }
    if (!changed) break;
  }

  Graph* copy = new Graph(g->op_registry());
  CopyGraph(*g, copy);
  delete g;
  *graph = copy;
  DumpGraph("ReCopy", *graph);
}
```

### Graph的执行

每个子图执行时，会调用前面设置的executor的`Run`\/`RunAsync`方法

```cpp
// tensorflow/core/common_runtime/executor.h
// Executor runs a graph computation.
// Example:
//   Graph* graph = ...;
//      ... construct graph ...
//   Executor* executor;
//   TF_CHECK_OK(NewSimpleExecutor(my_device, graph, &executor));
//   Rendezvous* rendezvous = NewNaiveRendezvous();
//   TF_CHECK_OK(rendezvous->Send("input", some_input_tensor));
//   TF_CHECK_OK(executor->Run({ExecutorOpts, rendezvous, nullptr}));
//   TF_CHECK_OK(rendezvous->Recv("input", &output_tensor));
//   ... ...
//
// Multiple threads can call Executor::Run concurrently.
```

`RunAsync`方法的实现

```cpp
// tensorflow/core/common_runtime/executor.cc
void ExecutorState::RunAsync(Executor::DoneCallback done) {
  const Graph* graph = impl_->graph_;
  TaggedNodeSeq ready;

  // Ask the device to fill in the device context map.
  Device* device = impl_->params_.device;
  Status fill_status = device->FillContextMap(graph, &device_context_map_);
  if (!fill_status.ok()) {
    done(fill_status);
    return;
  }

  // Initialize the ready queue.
  for (const Node* n : impl_->root_nodes_) {
    DCHECK_EQ(n->in_edges().size(), 0);
    ready.push_back(TaggedNode{n, root_frame_, 0, false});
  }
  if (ready.empty()) {
    done(Status::OK());
  } else {
    num_outstanding_ops_ = ready.size();
    root_frame_->iterations[0]->outstanding_ops = ready.size();
    done_cb_ = done;
    // Schedule to run all the ready ops in thread pool.
    ScheduleReady(ready, nullptr);
  }
}

void ExecutorState::ScheduleReady(const TaggedNodeSeq& ready,
                                  TaggedNodeReadyQueue* inline_ready) {
  if (ready.empty()) return;

  int64 scheduled_usec = 0;
  if (stats_collector_) {
    scheduled_usec = nodestats::NowInUsec();
  }
  if (inline_ready == nullptr) {
    // Schedule to run all the ready ops in thread pool.
    for (auto& tagged_node : ready) {
      runner_([=]() { Process(tagged_node, scheduled_usec); });
    }
    return;
  }
  const NodeItem* nodes = impl_->nodes_;
  const TaggedNode* curr_expensive_node = nullptr;
  for (auto& tagged_node : ready) {
    const NodeItem& item = nodes[tagged_node.node->id()];
    if (tagged_node.is_dead || !item.kernel_is_expensive) {
      // Inline this inexpensive node.
      inline_ready->push_back(tagged_node);
    } else {
      if (curr_expensive_node) {
        // Dispatch to another thread since there is plenty of work to
        // do for this thread.
        runner_(std::bind(&ExecutorState::Process, this, *curr_expensive_node,
                          scheduled_usec));
      }
      curr_expensive_node = &tagged_node;
    }
  }
  if (curr_expensive_node) {
    if (inline_ready->empty()) {
      // Tail recursion optimization
      inline_ready->push_back(*curr_expensive_node);
    } else {
      // There are inline nodes to run already. We dispatch this expensive
      // node to other thread.
      // 会在线程池中执行，runner_在direct_session.cc中赋值
      runner_(std::bind(&ExecutorState::Process, this, *curr_expensive_node,
                        scheduled_usec));
    }
  }
}

void ExecutorState::Process(TaggedNode tagged_node, int64 scheduled_usec) {
  // ...
      if (item.kernel_is_async) {
        // ...
        if (stats) nodestats::SetOpStart(stats);
        device->ComputeAsync(async, &state->ctx, done);
      } else {
        // Synchronous computes.
        OpKernelContext ctx(&params, item.num_outputs);
        if (stats) nodestats::SetOpStart(stats);
        device->Compute(CHECK_NOTNULL(op_kernel), &ctx);
        if (stats) nodestats::SetOpEnd(stats);

        s = ProcessOutputs(item, &ctx, &outputs, stats);
        if (s.ok() && impl_->device_record_tensor_accesses_) {
          // Get the list of all tensors accessed during the execution
          ctx.retrieve_accessed_tensors(&accessed_tensors);
          device_context = ctx.op_device_context();
        }
        if (stats) nodestats::SetMemory(stats, &ctx);
      }
  // ...
}
```

可以看到，在`ExecutorState::Process`调用了对应设备的Compute。而设备的Compute则调用了OpKernel的Compute方法\(此处多一层封装，主要是不同的设备在执行前会执行初始化代码\)。

* Q：ScheduleReady中没有循环逻辑如何处理整个graph？
* A：实际上ScheduleReady逻辑上是一个递归调用\(详细过程查看tensorflow\/core\/common\_runtime\/executor.cc\)
  * ScheduleReady
    * Process\(在线程池中执行\)
      * 执行kernel计算
      * PropagateOutputs\(根据outputs添加ready节点\)
      * NodeDone
        * ScheduleReady





### 线程池实现

前面Executor的runner中，实际的计算被提交到线程池中执行

```cpp
// tensorflow/core/common_runtime/direct_session.cc
void DirectSession::SchedClosure(thread::ThreadPool* pool,
                                 std::function<void()> c) {
// TODO(sanjay): Get rid of __ANDROID__ path
#ifdef __ANDROID__
  // On Android, there is no implementation of ThreadPool that takes
  // std::function, only Closure, which we cannot easily convert.
  //
  // Instead, we just run the function in-line, which is currently
  // safe given the reasoning above.
  c();
#else
  pool->Schedule(std::move(c));
#endif  // __ANDROID__
}
```

接下来，从ThreadPool实现可以看到，代码最终使用Eigen的线程池实现：

```cpp
// tensorflow/core/lib/core/threadpool.cc
ThreadPool::ThreadPool(Env* env, const ThreadOptions& thread_options,
                       const string& name, int num_threads) {
  CHECK_GE(num_threads, 1);
  impl_.reset(
      new ThreadPool::Impl(env, thread_options, "tf_" + name, num_threads));
}

struct ThreadPool::Impl : Eigen::ThreadPoolTempl<EigenEnvironment> {
  Impl(Env* env, const ThreadOptions& thread_options, const string& name,
       int num_threads)
      : Eigen::ThreadPoolTempl<EigenEnvironment>(
            num_threads, EigenEnvironment(env, thread_options, name)) {}

  void ParallelFor(int64 total, int64 cost_per_unit,
                   std::function<void(int64, int64)> fn) {
    CHECK_GE(total, 0);
    CHECK_EQ(total, (int64)(Eigen::Index)total);
    Eigen::ThreadPoolDevice device(this, this->NumThreads());
    device.parallelFor(
        total, Eigen::TensorOpCost(0, 0, cost_per_unit),
        [&fn](Eigen::Index first, Eigen::Index last) { fn(first, last); });
  }
};

void ThreadPool::Schedule(std::function<void()> fn) {
  CHECK(fn != nullptr);
  impl_->Schedule(std::move(fn));
}

void ThreadPool::ParallelFor(int64 total, int64 cost_per_unit,
                             std::function<void(int64, int64)> fn) {
  impl_->ParallelFor(total, cost_per_unit, std::move(fn));
}
```

