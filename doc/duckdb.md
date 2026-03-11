DuckDB Source、Sink、Operator 架构深度解析

DuckDB 的执行引擎采用 push-based pipeline 并行模型，其中算子被抽象为三种核心类型：Source、Operator 和 Sink。这种设计是 DuckDB 实现高性能并行查询的关键架构创新。

一、执行引擎演进：从 Volcano 到 Pipeline

1.1 传统 Volcano 模型的局限性

DuckDB 最初采用基于 pull-based 的 Vector Volcano 模型，但这种模型存在明显瓶颈：

模型 工作原理 缺点

Volcano 模型 自顶向下拉取数据，每个算子通过 GetNext() 方法向上游请求数据 1. 函数调用开销大<br>2. 对多线程并行支持差<br>3. 缓存局部性不佳

Pipeline 模型 自底向上推送数据，数据以管道形式流动 1. 更好的缓存利用<br>2. 天然支持并行<br>3. 减少虚函数调用

1.2 Push-based Pipeline 的优势

DuckDB 在 2021 年切换为 push-based 执行引擎，核心改进包括：
• Pipeline 并行：将执行计划拆分为多个 pipeline，每个 pipeline 内可并行执行

• 数据驱动：数据以 chunk（默认 1024 行）为单位在 pipeline 中流动

• 状态管理：Source 和 Sink 维护全局状态，Operator 通常无状态

二、三种算子的核心设计

2.1 Source 算子：数据生产者

Source 是 pipeline 的起点，负责从数据源读取数据并推入管道。
// Source 算子的核心接口
class PhysicalOperator {
public:
// 全局源状态（整个查询共享）
unique_ptr<GlobalSourceState> GetGlobalSourceState();

    // 本地源状态（每个线程独立）
    unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext &context);
    
    // 获取数据块
    SourceResultType GetData(ExecutionContext &context, 
                            DataChunk &chunk, 
                            GlobalSourceState &gstate, 
                            LocalSourceState &lstate);
};


Source 的关键特性：
• 并行度控制：通过 MaxThreads() 方法决定并行任务数

• 数据分区：将数据拆分为多个 morsel（小数据块）供不同线程处理

• 全局状态：协调多个线程的数据读取，避免重复或遗漏

常见的 Source 算子：
• PhysicalTableScan：表扫描

• PhysicalColumnDataScan：列数据扫描

• PhysicalCTEScan：CTE 扫描

• PhysicalExpressionScan：表达式扫描

2.2 Operator 算子：数据处理器

Operator 是 pipeline 的中间处理单元，执行具体的计算逻辑。
// Operator 的核心接口
class PhysicalOperator {
public:
// 执行操作
OperatorResultType Execute(ExecutionContext &context,
DataChunk &input,
DataChunk &chunk,
GlobalOperatorState &gstate,
OperatorState &state);
};


Operator 的关键特性：
• 无状态设计：大多数 Operator 不需要感知并行，每个线程独立处理

• 向量化计算：一次处理一个 DataChunk（1024 行），充分利用 SIMD

• 流水线执行：数据到达后立即处理，不等待完整数据集

常见的 Operator 算子：
• PhysicalFilter：数据过滤

• PhysicalProjection：列投影

• PhysicalHashJoin：哈希连接（Probe 端）

• PhysicalWindow：窗口函数

2.3 Sink 算子：数据消费者

Sink 是 pipeline 的终点，负责收集和处理最终结果。
// Sink 算子的核心接口
class PhysicalOperator {
public:
// 全局接收状态
unique_ptr<GlobalSinkState> GetGlobalSinkState();

    // 本地接收状态
    unique_ptr<LocalSinkState> GetLocalSinkState(ExecutionContext &context);
    
    // 处理输入数据（线程本地）
    SinkResultType Sink(ExecutionContext &context,
                       GlobalSinkState &gstate,
                       LocalSinkState &lstate,
                       DataChunk &input);
    
    // 合并本地状态到全局状态
    void Combine(ExecutionContext &context,
                GlobalSinkState &gstate,
                LocalSinkState &lstate);
    
    // 最终化处理
    void Finalize(ClientContext &context, GlobalSinkState &gstate);
};


Sink 的关键特性：
• 全局状态管理：协调多个线程的中间结果

• 两阶段聚合：Sink() 处理本地数据，Combine() 合并到全局状态

• 线程同步点：Sink 是 pipeline 中唯一需要线程同步的地方

常见的 Sink 算子：
• PhysicalHashAggregate：哈希聚合

• PhysicalOrder：排序

• PhysicalLimit：限制结果集

• PhysicalResultCollector：结果收集

三、Pipeline 并行执行模型

3.1 Pipeline 构建过程

DuckDB 将物理执行计划树转换为多个 pipeline：
// 简化的 pipeline 构建逻辑
void BuildPipelines(Pipeline &current,
vector<shared_ptr<Pipeline>> &pipelines) {
if (IsSinkOperator(this)) {
// 创建新的 pipeline，当前算子作为 sink
auto new_pipeline = make_shared<Pipeline>();
new_pipeline->sink = shared_from_this();
pipelines.push_back(new_pipeline);

        // 递归构建子 pipeline
        for (auto &child : children) {
            child->BuildPipelines(*new_pipeline, pipelines);
        }
    } else {
        // 将当前算子添加到 pipeline
        current.operators.push_back(shared_from_this());
        
        // 继续构建
        for (auto &child : children) {
            child->BuildPipelines(current, pipelines);
        }
    }
}


3.2 并行执行架构


执行计划示例：
PROJECTION
|
HASH_JOIN
/    \
TABLE_SCAN  TABLE_SCAN

转换为 pipeline：
Pipeline 1 (Build端):
Source: TABLE_SCAN (右表)
Sink: HASH_JOIN (构建哈希表)

Pipeline 2 (Probe端):
Source: TABLE_SCAN (左表)
Operators: HASH_JOIN (探测), PROJECTION
Sink: RESULT_COLLECTOR


3.3 Morsel-Driven 并行

DuckDB 采用 Morsel-Driven 并行框架，核心思想：
1. 数据分片：将输入数据划分为小的 morsel
2. 工作窃取：线程动态获取 morsel 处理
3. 负载均衡：避免数据倾斜导致的性能问题
   // 并行任务调度
   void ScheduleTasks() {
   // 根据 Source 的 MaxThreads() 决定并行度
   idx_t num_threads = source->MaxThreads();

   vector<unique_ptr<Task>> tasks;
   for (idx_t i = 0; i < num_threads; i++) {
   tasks.push_back(make_unique<PipelineTask>(pipeline, context));
   }

   // 提交到任务调度器
   task_scheduler->Schedule(tasks);
   }


四、核心算子实现解析

4.1 排序算子 (PhysicalOrder) 的 Sink 实现

// 排序算子的 Sink 接口实现
SinkResultType PhysicalOrder::Sink(ExecutionContext &context,
GlobalSinkState &gstate_p,
LocalSinkState &lstate_p,
DataChunk &input) const {
auto &lstate = (OrderLocalSinkState &)lstate_p;
auto &gstate = (OrderGlobalSinkState &)gstate_p;

    // 1. 将列式数据转换为行式（排序需要）
    local_sort_state.SinkChunk(keys, payload);
    
    // 2. 达到内存阈值时进行局部排序
    if (local_sort_state.SizeInBytes() >= gstate.memory_per_thread) {
        local_sort_state.Sort(global_sort_state, true);
    }
    
    return SinkResultType::NEED_MORE_INPUT;
}

// Combine 接口：合并各线程的排序结果
void PhysicalOrder::Combine(ExecutionContext &context,
GlobalSinkState &gstate_p,
LocalSinkState &lstate_p) const {
auto &gstate = (OrderGlobalSinkState &)gstate_p;
auto &lstate = (OrderLocalSinkState &)lstate_p;

    // 1. 对剩余数据进行排序
    local_sort_state.Sort(*this, external || !local_sort_state.sorted_blocks.empty());
    
    // 2. 加锁，将本地排序结果合并到全局状态
    lock_guard<mutex> append_guard(lock);
    for (auto &sb : local_sort_state.sorted_blocks) {
        sorted_blocks.push_back(move(sb));
    }
}


4.2 聚合算子 (PhysicalPerfectHashAggregate) 的 Sink 实现

SinkResultType PhysicalPerfectHashAggregate::Sink(
ExecutionContext &context,
GlobalSinkState &state,
LocalSinkState &lstate,
DataChunk &input) const {

    // 1. 分离 Group Chunk（聚合键）和 Aggregate Input Chunk（聚合值）
    DataChunk group_chunk, aggregate_input_chunk;
    SplitInputChunk(input, group_chunk, aggregate_input_chunk);
    
    // 2. 在本地状态上执行聚合
    auto &local_state = (PerfectHashAggregateLocalState &)lstate;
    local_state.Aggregate(group_chunk, aggregate_input_chunk);
    
    return SinkResultType::NEED_MORE_INPUT;
}


4.3 表扫描算子 (PhysicalTableScan) 的 Source 实现

SourceResultType PhysicalTableScan::GetData(
ExecutionContext &context,
DataChunk &chunk,
GlobalSourceState &gstate,
LocalSourceState &lstate) {

    auto &global_state = (TableScanGlobalState &)gstate;
    auto &local_state = (TableScanLocalState &)lstate;
    
    // 1. 获取下一个数据分区（morsel）
    auto morsel = global_state.GetNextMorsel(local_state.thread_id);
    if (!morsel) {
        return SourceResultType::FINISHED;
    }
    
    // 2. 从存储引擎读取数据块
    storage_manager.ReadMorsel(morsel, chunk);
    
    // 3. 应用谓词下推（如果存在）
    if (has_predicate) {
        ApplyPredicate(chunk);
    }
    
    return SourceResultType::HAVE_MORE_OUTPUT;
}


五、状态管理机制

5.1 全局状态 vs 本地状态

状态类型 生命周期 访问模式 典型用途

GlobalState 整个查询期间 多线程共享，需要同步 哈希表、排序缓冲区、结果收集器

LocalState 单个线程执行期间 线程私有，无需同步 线程本地缓存、中间计算结果

5.2 状态传递流程


查询开始
↓
创建 GlobalSourceState 和 GlobalSinkState
↓
为每个线程创建 LocalSourceState 和 LocalSinkState
↓
并行执行：
Source.GetData() → Operator.Execute() → Sink.Sink()
↓
所有线程完成后调用 Sink.Combine()
↓
Sink.Finalize() 生成最终结果


六、性能优化策略

6.1 向量化执行

// 向量化过滤示例
void VectorizedFilter(DataChunk &input, DataChunk &output, SelectionVector &sel) {
// 一次处理整个向量（1024行）
for (idx_t i = 0; i < input.size(); i++) {
if (predicate.Evaluate(input, i)) {
sel.set_index(output.size(), i);
output.size()++;
}
}

    // 批量复制选中行
    if (output.size() > 0) {
        input.Slice(output, sel, output.size());
    }
}


6.2 内存优化

• 数据块大小：默认 1024 行，平衡缓存利用和并行粒度

• 内存池：预分配内存，减少动态分配开销

• 零拷贝：在 pipeline 间传递数据时尽量共享内存

6.3 并行度自适应

idx_t PhysicalTableScan::MaxThreads() const {
// 基于数据大小和系统资源动态决定并行度
idx_t data_size = EstimateDataSize();
idx_t available_threads = TaskScheduler::GetScheduler(context).NumberOfThreads();

    // 每个线程至少处理 1MB 数据
    idx_t min_chunk_size = 1024 * 1024; // 1MB
    idx_t suggested_threads = data_size / min_chunk_size;
    
    return min(suggested_threads, available_threads);
}


七、实际应用示例

7.1 简单查询的执行流程

-- 示例查询
SELECT department, AVG(salary)
FROM employees
WHERE salary > 50000
GROUP BY department
ORDER BY AVG(salary) DESC
LIMIT 10;


对应的 Pipeline 结构：

Pipeline 1: 扫描和过滤
Source: PhysicalTableScan (employees)
Operator: PhysicalFilter (salary > 50000)
Sink: PhysicalHashAggregate (构建哈希表)

Pipeline 2: 聚合和排序
Source: Pipeline 1 的输出
Operator: PhysicalHashAggregate (计算 AVG)
Operator: PhysicalOrder (排序)
Operator: PhysicalLimit (取前10)
Sink: PhysicalResultCollector


7.2 复杂查询（Join）的 Pipeline 拆分

SELECT a.id, b.name
FROM table_a a
JOIN table_b b ON a.b_id = b.id
WHERE a.value > 100 AND b.category = 'A';


Pipeline 依赖关系：

MetaPipeline (根)
├── Pipeline 1: 构建哈希表
│   ├── Source: Scan table_b
│   ├── Operator: Filter (category = 'A')
│   └── Sink: HashJoin (构建端)
│
└── Pipeline 2: 探测和过滤
├── Source: Scan table_a
├── Operator: Filter (value > 100)
├── Operator: HashJoin (探测端)
└── Sink: ResultCollector


八、设计优势与局限性

8.1 优势

1. 高效并行：Source 决定并行度，Sink 处理同步，中间 Operator 无状态
2. 缓存友好：数据以 chunk 为单位在缓存中流动
3. 负载均衡：Morsel-Driven 框架自动平衡线程负载
4. 扩展性强：新算子只需实现相应接口即可集成

8.2 局限性

1. 状态管理复杂：Sink 算子需要处理多线程同步
2. 数据倾斜敏感：不均匀的数据分布可能影响并行效果
3. 内存压力：某些 Sink（如排序、哈希聚合）需要大量内存

九、最佳实践

9.1 开发新算子

// 实现新算子的步骤
class MyCustomOperator : public PhysicalOperator {
public:
// 1. 实现 Operator 接口
OperatorResultType Execute(...) override;

    // 2. 如果是 Source 或 Sink，实现相应接口
    unique_ptr<GlobalSourceState> GetGlobalSourceState() override;
    SourceResultType GetData(...) override;
    
    // 或
    unique_ptr<GlobalSinkState> GetGlobalSinkState() override;
    SinkResultType Sink(...) override;
    void Combine(...) override;
    void Finalize(...) override;
    
    // 3. 注册到执行引擎
    static void Register() {
        OperatorRegistry::Register<MyCustomOperator>();
    }
};


9.2 性能调优建议

1. 选择合适的并行度：根据数据大小和硬件资源调整
2. 监控内存使用：关注 Sink 算子的内存消耗
3. 避免数据倾斜：在可能的情况下均匀分布数据
4. 利用向量化：确保算子实现充分利用 SIMD 指令

十、总结

DuckDB 的 Source、Sink、Operator 架构是其高性能执行引擎的核心：

1. 关注点分离：Source 负责数据生产，Operator 负责数据处理，Sink 负责数据消费
2. 并行友好：Source 控制并行度，Sink 处理同步，中间算子无状态
3. 状态管理：GlobalState 用于跨线程协调，LocalState 用于线程本地计算
4. 流水线执行：数据驱动，减少等待时间，提高 CPU 利用率

这种设计使 DuckDB 能够在单机上实现接近分布式的查询性能，特别适合 OLAP 工作负载。通过将复杂查询拆分为多个 pipeline，DuckDB 能够充分利用多核 CPU 和现代硬件特性，在嵌入式环境中提供卓越的分析性能。

对于开发者而言，理解这三种算子的角色和交互方式，是优化 DuckDB 查询性能、开发自定义扩展的关键。无论是调整现有算子的行为，还是实现全新的数据处理逻辑，都需要遵循这一架构模式，确保与执行引擎的无缝集成。