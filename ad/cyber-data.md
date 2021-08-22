## Data目录
* buffer 与不同 channel 融合
首先定义了cache buffer，实际就是一个循环数组做缓冲区，满了会覆盖最旧的。另外icache buffer可以注册一个fusion函数，覆盖掉这套循环缓冲的逻辑。
AllLatest实现了一套简单的融合逻辑，也就是以M0的消息为准，每次M0有新的消息，执行这个自定义回调，读取M1-M3的最新消息，得到一个融合后的复合buffer。
这里**没看到一些复杂的等待逻辑**
```c++
  // data_visitor.h
  bool TryFetch(std::shared_ptr<M0>& m0, std::shared_ptr<M1>& m1) {  // NOLINT
    // Fusion命名不太好，实际是FetchFusionedMessage
    if (data_fusion_->Fusion(&next_msg_index_, m0, m1)) {
      next_msg_index_++;
      return true;
    }
    return false;
  }
```
* DataDispatcher：就是定义了一个简单的thread-safe map，每个channel id对应一个channel buffer (里面是cache buffer)的俩表，
代表每个channel的消费者的buffer。生产者可以调用`Dispatch`进行分发，同时调用一个notifier。注意这个notifier在协程创建时注册的回调。
```c++
template <typename T>
bool DataDispatcher<T>::Dispatch(const uint64_t channel_id,
                                 const std::shared_ptr<T>& msg) {
  BufferVector* buffers = nullptr;
  if (apollo::cyber::IsShutdown()) {
    return false;
  }
  if (buffers_map_.Get(channel_id, &buffers)) {
    for (auto& buffer_wptr : *buffers) {
      if (auto buffer = buffer_wptr.lock()) {
        std::lock_guard<std::mutex> lock(buffer->Mutex());
        buffer->Fill(msg);
      }
    }
  } else {
    return false;
  }
  return notifier_->Notify(channel_id);
}
```

```c++
// scheduler.cc
bool Scheduler::CreateTask(std::function<void()>&& func,
                           const std::string& name,
                           std::shared_ptr<DataVisitorBase> visitor) {
  if (cyber_unlikely(stop_.load())) {
    ADEBUG << "scheduler is stoped, cannot create task!";
    return false;
  }

  auto task_id = GlobalData::RegisterTaskName(name);

  auto cr = std::make_shared<CRoutine>(func);
  cr->set_id(task_id);
  cr->set_name(name);
  AINFO << "create croutine: " << name;

  if (!DispatchTask(cr)) {
    return false;
  }

  if (visitor != nullptr) {
    visitor->RegisterNotifyCallback([this, task_id]() {
      if (cyber_unlikely(stop_.load())) {
        return;
      }
      this->NotifyProcessor(task_id);
    });
  }
  return true;
}
```
`NotifyProcessor`会修改协程的运行状态？？？：
```c++
bool SchedulerClassic::NotifyProcessor(uint64_t crid) {
  if (cyber_unlikely(stop_)) {
    return true;
  }

  {
    ReadLockGuard<AtomicRWLock> lk(id_cr_lock_);
    if (id_cr_.find(crid) != id_cr_.end()) {
      auto cr = id_cr_[crid];
      if (cr->state() == RoutineState::DATA_WAIT ||
          cr->state() == RoutineState::IO_WAIT) {
        cr->SetUpdateFlag();
      }

      ClassicContext::Notify(cr->group_name());
      return true;
    }
  }
  return false;
}
```
