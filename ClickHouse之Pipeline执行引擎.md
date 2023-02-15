---
title: ClickHouse之Pipeline执行引擎
date: 2023-02-19 18:37:35
tags: clickhouse
---
原文转发自知乎叶绿素: https://zhuanlan.zhihu.com/p/545776764

## 目录

- 核心概念：IProcessor和Port（ISink、ISource、IXXXTransform、InputPort、OutputPort...）**未完成**
- pipeline的组装：QueryPlan、IQueryPlanStep、QueryPipelineBuilders、Pipe、QueryPipeline **未完成**
- pipeline的执行：PipelineExecutor、push模型和pull模型 **本篇**
- pipeline的特点：支持运行时扩展（不单独分析，参考[这篇文章](https://zhuanlan.zhihu.com/p/591575835)）、支持与网络io结合 **本篇**

## 关于PipelineExecutor

位置:src/Processors/Executors/PipelineExecutor.h

代表query pipeline的调度器，主要数据成员：

ExecutingGraph(graph)；

- pipeline在执行前，会被执行器转换成一个ExecutingGraph，由一组Node和对应的Edge集合组成，Node代表单个的Processor，Edge代表连接着OutputPort和InputPort的对象。每个Node持有两条边，分别为direct_edges和back_edges，direct_edges代表本Node的outputport连接其他inputport，back_edges代表某个outputport连接本Node的inputport。

ExecutorTasks(tasks)；

- 管理可以被调度执行的任务，主要数据成员如下：

1. executor_contexts;（work线程上下文）
2. task_queue;（可以被调度执行的Node集合）
3. threads_queue;（由于缺少任务被挂起的线程id集合）

调用链：

```cpp
PipelineExecutor::execute(size_t num_threads)
-> executeImpl(num_threads);
  -> initializeExecution(num_threads);
    -> graph->initializeExecution(queue);
    // 将所有direct_edges为空的Node提取出来，依次调用updateNode。
      -> updateNode(proc, queue, async_queue);               （2）
     // 以proc为起点，调用prepare函数，如果遇到Ready的node则push到queue中，  如果遇到Async的node则push到async_queue中。然后更新与当前node关联的edge，  通过edge找到下一个node并执行该node的prepare方法，最终将所有状态为Ready      和Async的node放到对应的队列中。
    -> tasks.init();
    -> tasks.fill(queue);
    //tasks持有线程队列和任务队列（任务队列是个二维数组，及为每个线程维护一个任务队列，详情见TaskQueue类），tasks.fill(queue)实际上就是将就绪的node依次分配到任务队列中。
   -> threads.emplace_back(executeSingleThread);
   //在栈上创建一个线程池，依次调用executeSingleThread函数：
     -> executeStepImpl(thread_num);
       -> auto& context = tasks.getThreadContext(thread_num);
       -> tasks.tryGetTask(context);
       // 首先尝试从本上下文中获取异步task，如果没有的话尝试从task_queue中获取一个task。
       // 如果task_queue为空并且async_task也为空的话，则将context.thread_number插入thread_queue（这个对象中记录着当前等待着task的线程id）中，并wait在wakeup_flag || finished上；如果context取到了task，则调用tryWakeupAnyOtherThreadWithTasks函数。
         -> tryWakeupAnyOtherThreadWithTasks(self, lock);
         // 这个函数的目的在于获取一个等待任务的线程进行唤醒：executor_contexts[thread_to_wake]->wakeUp();
       -> context.executeTask();
         -> executeJob(node, read_progress_callback);
           -> node->processor->work();
         -> graph->updateNode(context.getProcessorID(), queue, async_queue);  （1）
         -> tasks.pushTasks(queue, async_queue, context);
         // 将queue中的node插入task_queue，并唤醒其他线程来处理task
           -> tryWakeupAnyOtherThreadWithTasks(context, lock);
```

总结PipelineExecutor的调度流程：

从ExecutingGraph中获取direct_edges为空的Node并调用其prepare函数。direct_edges为空的Node一般来说就是ISink类型的节点，即最终的消费者。可以在ISink函数中看到其prepare函数的实现，首次调用时总是会返回Ready，因此调度器会调用其work函数，对ISink对象首次调用work函数会触发OnStart回调。

work函数调用之后，调度器会对该节点调用updateNode函数（见（1）），updateNode具体逻辑在（2）这里，即再次调用其prepare函数。这时ISink会调用input.setNeed函数，这个函数会唤醒对应的Edge（updated_edges.push(edge)），在updateNode逻辑中会处理这些Edge，获取对应的Node继续prepare操作。

因此，可以根据Edge关系从ISink节点出发，一直找到ISource节点并调用其prepare函数，对于ISource节点来说只要output_port可以push就返回Ready，由调度器调用work函数，work函数中执行tryGenerate函数（真正生成数据的函数）。因此当调度器再次执行其prepare函数时，执行output.pushData函数，这个函数和input.setNeed同样会唤醒对应的Edge，因此调度器会找到其下游节点调用prepare函数，这时数据已经从ISource节点交付，因此下游节点会返回Ready，调度器调用其work函数...从上游节点到最终的ISink节点重复这个操作。

最后我们会回到ISink节点，调用其work函数，work函数中会调用consume函数消费数据。当再次调用ISink节点的prepare函数时，会再次调用input.setNeed函数，这样就形成了一个循环。

可以看到，PipelineExecutor是一个pull模型的调度器，我们每次总ISink节点开始向上游节点请求数据，通过唤醒Edge将请求传递到ISource节点，在ISource节点中生产数据，交付下游节点处理，最终回到ISink节点消费数据，如果还有数据需要消费的话，ISink节点会再次向上游节点请求；当数据消费完成后，ISource节点会通过关闭output_port通知下游节点，最终完成所有数据的处理。

## 封装PipelineExecutor

在ClickHouse中并不是直接通过PipelineExecutor进行调度，而是将其封装在PullingAsyncPipelineExecutor、PullingPipelineExecutor、PushingAsyncPipelineExecutor、PushingPipelineExecutor等中进一步封装使用。其中PullingAsyncPipelineExecutor和PullingPipelineExecutor是pull模型的调度器，其区别在于是在当前线程调度还是申请一个线程进行调度。这两个调度器对PipelineExecutor的封装比较浅，原因如上，PipelineExecutor也是pull模型。而PushingAsyncPipelineExecutor和PushingPipelineExecutor是push模型的调度器，接下来以PushingAsyncPipelineExecutor为例进行分析，说明这是如何实现的。

在PushingAsyncPipelineExecutor中定义了一个ISource子类型PushingAsyncSource：

```text
class PushingAsyncSource : public ISource
```

在构造函数中创建了一个PushingAsyncSource的对象pushing_source，并将其连接到pipeline的input上：

```text
PushingAsyncPipelineExecutor::PushingAsyncPipelineExecutor(QueryPipeline & pipeline_) : pipeline(pipeline_)
{
 if (!pipeline.pushing())
   throw Exception(ErrorCodes::LOGICAL_ERROR, "Pipeline for PushingPipelineExecutor must be pushing");

 pushing_source = std::make_shared<PushingAsyncSource>(pipeline.input->getHeader());
 connect(pushing_source->getPort(), *pipeline.input);
 pipeline.processors.emplace_back(pushing_source);
}
```

PushingAsyncPipelineExecutor的start函数如下，可以看到没有什么特别的地方，构造一个PipelineExecutor，并在threadFunction中调用execute()函数：

```text
void PushingAsyncPipelineExecutor::start()
{
 if (started)
   return;

 started = true;

 data = std::make_unique<Data>();
 data->executor = std::make_shared<PipelineExecutor>(pipeline.processors, pipeline.process_list_element);
 data->executor->setReadProgressCallback(pipeline.getReadProgressCallback());
 data->source = pushing_source.get();

 auto func = [&, thread_group = CurrentThread::getGroup()]()
 {
   threadFunction(*data, thread_group, pipeline.getNumThreads());
 };

 data->thread = ThreadFromGlobalPool(std::move(func));
}
```

每次在push数据时，将数据传入pushing_source：

```text
void PushingAsyncPipelineExecutor::push(Chunk chunk)
{
 if (!started)
   start();

 bool is_pushed = pushing_source->setData(std::move(chunk));
 data->rethrowExceptionIfHas();

 if (!is_pushed)
   throw Exception(ErrorCodes::LOGICAL_ERROR,
     "Pipeline for PushingPipelineExecutor was finished before all data was inserted");
}
```

接下来看看PushingAsyncSource的定义，注意在generate()中等待在条件变量condvar上，只有当setData()或者finish()时才会唤醒generate()继续执行。所以，当PipelineExecutor调度时，首先从ISink一路请求数据到ISource，ISource的work函数会等待用户push数据，拿到数据后将数据传递给下游节点处理，最终到达ISink。如果还没有结束，ISink会重复这一逻辑，即再次一路请求数据到ISource并等待用户push数据。这样就利用pull模型的PipelineExecutor实现了push模型的调度器。

```text
class PushingAsyncSource : public ISource
{
public:
 explicit PushingAsyncSource(const Block & header)
        : ISource(header)
    {}

 String getName() const override { return "PushingAsyncSource"; }

 bool setData(Chunk chunk)
 {
   std::unique_lock lock(mutex);
   condvar.wait(lock, [this] { return !has_data || is_finished; });

   if (is_finished)
     return false;

   data.swap(chunk);
   has_data = true;
   condvar.notify_one();

   return true;
 }

 void finish()
 {
   std::unique_lock lock(mutex);
   is_finished = true;
   condvar.notify_all();
  }

protected:
 Chunk generate() override
 {
   std::unique_lock lock(mutex);
   condvar.wait(lock, [this] { return has_data || is_finished; });
   Chunk res;

   res.swap(data);
   has_data = false;
   condvar.notify_one();
   return res;
 }

private:
 Chunk data;
 bool has_data = false;
 bool is_finished = false;
 std::mutex mutex;
 std::condition_variable condvar;
};
```

push模型调度器在TCPHandler::processInsertQuery()函数中调用，用于处理Insert语句，而pull模型调度器在TCPHandler::processOrdinaryQueryWithProcessors()函数中调用，处理非Insert语句：

```text
/// Processing Query
state.io = executeQuery(state.query, query_context, false, state.stage);

after_check_cancelled.restart();
after_send_progress.restart();

if (state.io.pipeline.pushing())
/// FIXME: check explicitly that insert query suggests to receive data via native protocol,
{
    state.need_receive_data_for_insert = true;
    processInsertQuery();
 }
 else if (state.io.pipeline.pulling())
 {
    processOrdinaryQueryWithProcessors();
 }
```

## Status::Async，执行引擎与网络io的无缝衔接

观察prepare函数返回的枚举类型，其中前四种状态与数据传递相关，后两种比较特别：ExpandPipeline用来支持运行时扩展pipeline，Async用来支持网络io。

```cpp
    enum class Status
    {
        /// Processor needs some data at its inputs to proceed.
        /// You need to run another processor to generate required input and then call 'prepare' again.
        NeedData,

        /// Processor cannot proceed because output port is full or not isNeeded().
        /// You need to transfer data from output port to the input port of another processor and then call 'prepare' again.
        PortFull,

        /// All work is done (all data is processed or all output are closed), nothing more to do.
        Finished,

        /// No one needs data on output ports.
        /// Unneeded,

        /// You may call 'work' method and processor will do some work synchronously.
        Ready,

        /// You may call 'schedule' method and processor will return descriptor.
        /// You need to poll this descriptor and call work() afterwards.
        Async,

        /// Processor wants to add other processors to pipeline.
        /// New processors must be obtained by expandPipeline() call.
        ExpandPipeline,
    };
```

搜一下ClickHouse源码，发现状态Async用在**RemoteSource**这个ISource类中，这个类用来支持需要通过网络io获取数据源的情况，比如分布式表（StorageDistributed）、支持S3（StorageS3Cluster）、支持HDFS（StorageHDFSCluster）等情况。

### Pipeline执行引擎的视角

首先从Pipeline执行引擎的视角开始，当节点node的prepare函数返回Status::Async时，该节点被push进async_queue中（updateNode内）：

```cpp
                    case IProcessor::Status::Async:
                    {
                        node.status = ExecutingGraph::ExecStatus::Executing;
                        async_queue.push(&node);
                        break;
                    }
```

当updateNode函数返回后，调用tasks.pushTasks函数，将由于本次调度产生的新任务push到全局队列中：

```cpp
            /// Try to execute neighbour processor.
            {
                Queue queue;
                Queue async_queue;

                /// Prepare processor after execution.
                if (!graph->updateNode(context.getProcessorID(), queue, async_queue))
                    finish();

                /// Push other tasks to global queue.
                tasks.pushTasks(queue, async_queue, context);
            }
```

------

```text
void ExecutorTasks::pushTasks(Queue & queue, Queue & async_queue, ExecutionThreadContext & context)
{
    context.setTask(nullptr);

    /// Take local task from queue if has one.
    if (!queue.empty() && !context.hasAsyncTasks())
    {
        context.setTask(queue.front());
        queue.pop();
    }

    if (!queue.empty() || !async_queue.empty())
    {
        std::unique_lock lock(mutex);

#if defined(OS_LINUX)
        while (!async_queue.empty() && !finished)
        {
            int fd = async_queue.front()->processor->schedule();
            async_task_queue.addTask(context.thread_number, async_queue.front(), fd);
            async_queue.pop();
        }
#endif

        while (!queue.empty() && !finished)
        {
            task_queue.push(queue.front(), context.thread_number);
            queue.pop();
        }

        /// Wake up at least one thread that will wake up other threads if required
        tryWakeUpAnyOtherThreadWithTasks(context, lock);
    }
}
```

关注async_queue部分，通过调用节点的schedule拿到fd，然后调用async_task_queue.addTask函数：

```cpp
void PollingQueue::addTask(size_t thread_number, void * data, int fd)
{
    std::uintptr_t key = reinterpret_cast<uintptr_t>(data);
    if (tasks.contains(key))
        throw Exception(ErrorCodes::LOGICAL_ERROR, "Task {} was already added to task queue", key);

    tasks[key] = TaskData{thread_number, data, fd};
    epoll.add(fd, data);
}
```

这个函数实际上就是记录注册节点，并通过epoll监听fd。看来这部分是注册流程，那么epoll是在哪里启动的，当fd就绪时又是如何通知工作线程进行调度的呢？接着看：

```cpp
void PipelineExecutor::executeImpl(size_t num_threads)
{
    initializeExecution(num_threads);

    bool finished_flag = false;

    SCOPE_EXIT_SAFE(
        if (!finished_flag)
        {
            finish();
            joinThreads();
        }
    );

    if (num_threads > 1)
    {
        spawnThreads(); // start at least one thread
        tasks.processAsyncTasks();
        joinThreads();
    }
    else
    {
        auto slot = slots->tryAcquire();
        executeSingleThread(0);
    }

    finished_flag = true;
}
```

在启动流程中，spawnThreads()函数启动多个工作线程，之后调用了tasks.processAsyncTasks()：

```cpp
void ExecutorTasks::processAsyncTasks()
{
#if defined(OS_LINUX)
    {
        /// Wait for async tasks.
        std::unique_lock lock(mutex);
        while (auto task = async_task_queue.wait(lock))
        {
            auto * node = static_cast<ExecutingGraph::Node *>(task.data);
            executor_contexts[task.thread_num]->pushAsyncTask(node);
            ++num_waiting_async_tasks;

            if (threads_queue.has(task.thread_num))
            {
                threads_queue.pop(task.thread_num);
                executor_contexts[task.thread_num]->wakeUp();
            }
        }
    }
#endif
}
```

这个函数的工作很简单，不断通过epoll_wait获取就绪的 fd对应的节点，并将该节点push到对应线程的上下文中，并且唤醒对应线程处理。**工作线程总是优先处理本上下文中的异步任务，如果没有才会从全局队列中获取任务，**见tasks.tryGetTask函数。

总结一下：

- 调度引擎启动时，启动多个工作线程，并在主线程通过Epoll监听异步任务。
- 当一个节点等待网络io时，在prepare函数中返回Status::Async，并在schedule函数中返回fd。
- 调度引擎拿到fd注册到Epoll中，当fd就绪时，主线程负责将对应的异步任务交付给对应的工作线程的上下文，并唤醒该工作线程。
- 工作线程每次获取任务时总是优先获取异步任务，拿到任务调用work函数。

### RemoteSource（选读）

现在看下RemoteSource节点的实现：

```cpp
std::optional<Chunk> RemoteSource::tryGenerate()
{
    /// onCancel() will do the cancel if the query was sent.
    if (was_query_canceled)
        return {};

    if (!was_query_sent)
    {
        /// Progress method will be called on Progress packet.
        query_executor->setProgressCallback([this](const Progress & value)
        {
            if (value.total_rows_to_read)
                addTotalRowsApprox(value.total_rows_to_read);
            progress(value.read_rows, value.read_bytes);
        });

        /// Get rows_before_limit result for remote query from ProfileInfo packet.
        query_executor->setProfileInfoCallback([this](const ProfileInfo & info)
        {
            if (rows_before_limit && info.hasAppliedLimit())
                rows_before_limit->set(info.getRowsBeforeLimit());
        });

        query_executor->sendQuery();

        was_query_sent = true;
    }

    Block block;

    if (async_read)
    {
        auto res = query_executor->read(read_context);
        if (std::holds_alternative<int>(res))
        {
            fd = std::get<int>(res);
            is_async_state = true;
            return Chunk();
        }

        is_async_state = false;

        block = std::get<Block>(std::move(res));
    }
    else
        block = query_executor->read();

    if (!block)
    {
        query_executor->finish(&read_context);
        return {};
    }

    UInt64 num_rows = block.rows();
    Chunk chunk(block.getColumns(), num_rows);

    if (add_aggregation_info)
    {
        auto info = std::make_shared<AggregatedChunkInfo>();
        info->bucket_num = block.info.bucket_num;
        info->is_overflows = block.info.is_overflows;
        chunk.setChunkInfo(std::move(info));
    }

    return chunk;
}
```

这个类主要封装了query_executor，在work函数（ISource类型的节点重载了work函数，通过tryGenerate函数产生数据）中发送请求，然后通过query_executor->read读取结果，这里可以是阻塞读或者非阻塞读，阻塞读就很简单了，从connection中读取数据然后组包返回，这里不展开，主要关注下非阻塞读。

```text
auto res = query_executor->read(read_context);
```

这里返回的结果是一个variant<int, block>， 如果本次非阻塞读到了数据，那么走阻塞读的流程，否则会返回connection对应的fd，并将is_async_state设置为TRUE，**在prepare函数中如果看到is_async_state为true就会返回Status::Async：**

```cpp
ISource::Status RemoteSource::prepare()
{
    /// Check if query was cancelled before returning Async status. Otherwise it may lead to infinite loop.
    if (was_query_canceled)
    {
        getPort().finish();
        return Status::Finished;
    }

    if (is_async_state)
        return Status::Async;

    Status status = ISource::prepare();
    /// To avoid resetting the connection (because of "unfinished" query) in the
    /// RemoteQueryExecutor it should be finished explicitly.
    if (status == Status::Finished)
    {
        query_executor->finish(&read_context);
        is_async_state = false;
    }
    return status;
}
```

接着看下query_executor->read(read_context)这个函数，这部分内容其实和调度引擎关系不大，没有特别需求的读者可以将其当做个黑盒子，跳过这部分内容。

ClickHouse使用第三方库*Poco*进行开发，其中包括网络库，下面的内容涉及**到通过一个有栈协程衔接了pipeline执行引擎和\*Poco::Net\*库\*。\***

```cpp
std::variant<Block, int> RemoteQueryExecutor::read(std::unique_ptr<ReadContext> & read_context [[maybe_unused]])
{

#if defined(OS_LINUX)
    if (!sent_query)
    {
        sendQuery();

        if (context->getSettingsRef().skip_unavailable_shards && (0 == connections->size()))
            return Block();
    }

    if (!read_context || resent_query)
    {
        std::lock_guard lock(was_cancelled_mutex);
        if (was_cancelled)
            return Block();

        read_context = std::make_unique<ReadContext>(*connections);
    }

    do
    {
        if (!read_context->resumeRoutine())
            return Block();

        if (read_context->is_read_in_progress.load(std::memory_order_relaxed))
        {
            read_context->setTimer();
            return read_context->epoll.getFileDescriptor();
        }
        else
        {
            /// We need to check that query was not cancelled again,
            /// to avoid the race between cancel() thread and read() thread.
            /// (since cancel() thread will steal the fiber and may update the packet).
            if (was_cancelled)
                return Block();

            if (auto data = processPacket(std::move(read_context->packet)))
                return std::move(*data);
            else if (got_duplicated_part_uuids)
                return restartQueryWithoutDuplicatedUUIDs(&read_context);
        }
    }
    while (true);
#else
    return read();
#endif
}
```

主要关注ReadContext，首先调用read_context->resumeRoutine()，然后检测了一个flag：is_read_in_progress，如果这个flag为true，则说明本次非阻塞读没有读到数据，返回fd，否则就组包返回。

我们看下ReadContext的实现细节。

```cpp
RemoteQueryExecutorReadContext::RemoteQueryExecutorReadContext(IConnections & connections_)
    : connections(connections_)
{

    if (-1 == pipe2(pipe_fd, O_NONBLOCK))
        throwFromErrno("Cannot create pipe", ErrorCodes::CANNOT_OPEN_FILE);

    {
        epoll.add(pipe_fd[0]);
    }

    {
        epoll.add(timer.getDescriptor());
    }

    auto routine = RemoteQueryExecutorRoutine{connections, *this};
    fiber = boost::context::fiber(std::allocator_arg_t(), stack, std::move(routine));
}
```

构造函数中构造了一个fiber协程RemoteQueryExecutorRoutine：

```cpp
struct RemoteQueryExecutorRoutine
{
    IConnections & connections;
    RemoteQueryExecutorReadContext & read_context;

    struct ReadCallback
    {
        RemoteQueryExecutorReadContext & read_context;
        Fiber & fiber;

        void operator()(int fd, Poco::Timespan timeout = 0, const std::string fd_description = "")
        {
            try
            {
                read_context.setConnectionFD(fd, timeout, fd_description);
            }
            catch (DB::Exception & e)
            {
                e.addMessage(" while reading from {}", fd_description);
                throw;
            }

            read_context.is_read_in_progress.store(true, std::memory_order_relaxed);
            fiber = std::move(fiber).resume();
            read_context.is_read_in_progress.store(false, std::memory_order_relaxed);
        }
    };

    Fiber operator()(Fiber && sink) const
    {
        try
        {
            while (true)
            {
                read_context.packet = connections.receivePacketUnlocked(ReadCallback{read_context, sink}, false /* is_draining */);
                sink = std::move(sink).resume();
            }
        }
        catch (const boost::context::detail::forced_unwind &)
        {
            /// This exception is thrown by fiber implementation in case if fiber is being deleted but hasn't exited
            /// It should not be caught or it will segfault.
            /// Other exceptions must be caught
            throw;
        }
        catch (...)
        {
            read_context.exception = std::current_exception();
        }

        return std::move(sink);
    }
};
```

然后看下函数resumeRoutine()：

```cpp
bool RemoteQueryExecutorReadContext::resumeRoutine()
{
    if (is_read_in_progress.load(std::memory_order_relaxed) && !checkTimeout())
        return false;

    {
        std::lock_guard guard(fiber_lock);
        if (!fiber)
            return false;

        fiber = std::move(fiber).resume();

        if (exception)
            std::rethrow_exception(exception);
    }

    return true;
}
```

可以看到，每次调用resumeRoutine()，都是在推动这个协程继续执行，协程中的逻辑是：

```cpp
            while (true)
            {
                read_context.packet = connections.receivePacketUnlocked(ReadCallback{read_context, sink}, false /* is_draining */);
                sink = std::move(sink).resume();
            }
```

connections.receivePacketUnlocked的逻辑是：进行非阻塞读，如果读到数据就返回，否则调用ReadCallback。（这个逻辑很重要）

ReadCallback的逻辑是：拿到fd，并设置is_read_in_progress为TRUE，然后切回resumeRoutine函数执行。（这个逻辑也很重要）

下面总结一下这里的执行流程，首先看非阻塞读读到数据的情况：

- RemoteSource::tryGenerate() 调用
- query_executor->read(read_context) 调用
- read_context->resumeRoutine() 切换到协程RemoteQueryExecutorRoutine执行
- read_context.packet = connections.receivePacketUnlocked读到数据，切回read_context->resumeRoutine()执行
- 检查is_read_in_progress，为false，组包返回。
- RemoteSource拿到数据，传递给下游节点。

接着看下非阻塞读没读到数据的情况：

- RemoteSource::tryGenerate() 调用
- query_executor->read(read_context) 调用
- read_context->resumeRoutine() 切换到协程RemoteQueryExecutorRoutine执行
- read_context.packet = connections.receivePacketUnlocked没读到数据，调用ReadCallback
- ReadCallback中设置is_read_in_progress为true，切回read_context->resumeRoutine()执行
- 检查is_read_in_progress为true，返回fd
- RemoteSource拿到fd，设置is_async_state = false，下次prepare函数调用时返回Status::Async
- 执行引擎使用epoll检测fd，可读时将RemoteSource节点push到工作线程上下文并唤醒工作线程
- 工作线程从上下文中获得RemoteSource节点，调用work函数（tryGenerate）
- query_executor->read(read_context) 调用
- read_context->resumeRoutine() 切换到协程RemoteQueryExecutorRoutine执行，此时这个协程的suspend point实际上是在ReadCallback中，is_read_in_progress设置为false。
- connections继续非阻塞读，如果**还是没读到，会再次调用ReadCallback，重复上述流程。**一般情况这里是可读的，因为epoll检测到了fd上的可读事件。connections读到数据，receivePacketUnlocked函数返回，设置read_context.packet，然后切回read_context->resumeRoutine()执行
- 检查is_read_in_progress为false，调用processPacket(std::move(read_context->packet))组包返回
- RemoteSource拿到数据，传递给下游节点。

总结：这部分内容，你说它不重要吧，它衔接了执行引擎和网络库，使得执行引擎支持异步网络io；你说它重要吧，它又是一些胶水代码，为早期网络库的选择擦屁股，多了很多不必要的逻辑，大大增加了阅读和理解成本，还容易出错。

对于这种体量的项目，一些基础模块还是自己开发比较好，封装网络库的工作量并不会太大（仅项目使用，不需要为了通用性买太多单），也可以根据需求灵活的调整各种姿势。

### 参考

[Push还是Pull，这是个问题么？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/528514990)

[ClickHouse Query Execution Pipeline](https://link.zhihu.com/?target=https%3A//presentations.clickhouse.com/meetup24/5.%20Clickhouse%20query%20execution%20pipeline%20changes/%23clickhouse-query-execution-pipeline)

[ClickHouse源码阅读(0000 0100) —— ClickHouse是如何执行SQL的_B_e_a_u_tiful1205的博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/B_e_a_u_tiful1205/article/details/103537034)