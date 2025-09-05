# DAG 执行框架核心梳理

## 1. 设计思想（高层）

* **职责分离**

  * **DAG 构建层**：定义节点与依赖（`DAG / DAGNode / DependencyMixin`）。
  * **上下文运行态**：一次运行对应一个 `DAGContext`，负责输入/输出、共享数据、变量。
  * **执行器层**：递归执行节点，管理分支、日志、生命周期（`DefaultWorkflowRunner`）。

* **环境感知**

  * 通过 `DAGVar` 维护 **线程/协程本地 DAG 栈**，确保同步/异步环境一致获取“当前 DAG”。
  * 系统组件（`SystemApp`、`Executor`、`VariablesProvider`）可全局注入。

* **可扩展**

  * 基础节点抽象 `DAGNode` → 执行算子 `BaseOperator` → 分支算子 `BranchOperator`。
  * 变量 `DAGVariables` 可合并/覆盖，适应子 DAG、复用场景。

* **可观测与可视化**

  * 打印/图形化 DAG。
  * 运行链路追踪（`root_tracer`）、节点日志 ID，便于调试与监控。

---

## 2. 关键类与职责

### 2.1 `DependencyMixin`

* 封装拓扑操作：`set_upstream / set_downstream`。
* 运算符重载：

  * `A << B` → A 依赖 B。
  * `A >> B` → A 在 B 上游。
* 校验同一 DAG 内部依赖。

### 2.2 `DAGVar`

* **DAG 栈管理**：

  * 异步：`contextvars` 存 `deque`；
  * 线程：`threading.local()` 存 `deque`。
* **系统注入**：

  * 可全局设置/获取 `SystemApp`、`Executor`、`VariablesProvider` 等。

### 2.3 `DAGLifecycle`

* 定义生命周期钩子：`before_dag_run()`、`after_dag_end()`。
* 要求幂等，防止多次调用副作用。

### 2.4 `DAGNode`

* DAG 内基础节点，具备：

  * 上下游依赖维护。
  * 自动挂载到当前 DAG。
  * 分配唯一 `node_id`。
* 支持序列化检查。

### 2.5 `DAGVariables`

* 声明式变量系统：支持作用域、标签、用户/系统标识。
* `merge()`：按键组合去重合并，入参优先。
* `to_provider()`：懒创建并缓存 `StorageVariablesProvider`。

### 2.6 `DAGContext`

* **运行态容器**：

  * `node_to_outputs`：保存每个节点的 `TaskContext`。
  * `share_data`：全局 KV，带锁保护。
  * `dag_variables`：合并/继承变量。
* **任务级共享**：`save_task_share_data / get_task_share_data`。

### 2.7 `DAG`

* 管理 DAG 内所有节点与依赖。
* 构建根/叶/触发节点集合。
* 生命周期：保存/清理 `DAGContext`。
* 可视化：`print_tree / visualize_dag / show`。

### 2.8 `DefaultWorkflowRunner`

* **核心执行器**。
* `execute_workflow`：创建/复用上下文 → 运行前钩子 → `_execute_node` → 结束清理。
* `_execute_node`：

  * 递归执行上游。
  * 构造输入/任务上下文。
  * 调用 `node._run()`。
  * 保存输出。
  * 处理分支跳过。

---

## 3. DAG 执行流程

1. **进入 DAG 上下文**：

   ```python
   with DAG("demo") as dag:
       ...
   ```

2. **构建依赖**：用 `>>` / `<<` 设置上下游。

3. **可视化检查**：`dag.print_tree()`。

4. **准备执行器**：

   ```python
   runner = DefaultWorkflowRunner()
   dag_ctx = await runner.execute_workflow(end_node, call_data={...})
   ```

5. **运行时创建 `DAGContext`**：复用或新建，注入变量。

6. **运行前钩子**：`before_dag_run()`。

7. **执行节点 `_execute_node`**：

   * 递归跑上游。
   * 构造 Input/TaskContext。
   * 运行 `_run()` 并保存结果。

8. **结束收尾**：`after_dag_end()` 清理运行态。

---

## 4. 上下文与数据流

* **节点输出**：存放在 `TaskContext` → 汇总到 `dag_ctx.node_to_outputs`。
* **下游输入**：从上游 `TaskContext` 聚合到 `DefaultInputContext`。
* **共享数据**：全局 KV 或任务级 KV。
* **变量**：支持合并覆盖，懒加载 provider。
* **系统组件**：通过 `DAGVar` 注入。

---

## 5. 分支与跳过

* `BranchOperator` 在 `TaskContext.metadata["skip_node_names"]` 写入要跳过的下游节点名。
* Runner 根据这些信息：

  * 标记节点为跳过。
  * 如果节点支持 `can_skip_in_branch()`，则其下游整链路跳过。

---

## 6. 并发与生命周期

* **串行执行**：目前依赖递归串行（预留并行扩展点）。
* **同步/异步兼容**：同步任务可交给线程池 Executor。
* **生命周期**：

  * `before_dag_run()`。
  * `after_dag_end()`（可能多次，需幂等）。
* **流式执行**：若 `streaming_call=True`，跳过收尾清理，支持流式产出。

---

## 7. 可视化与调试

* **DAG 可视化**：`print_tree / visualize_dag / show`。
* **追踪日志**：

  * 节点执行带 `log_index`。
  * `root_tracer.start_span()` 打点链路。

---

## 8. 错误与日志

* 节点执行异常：捕获 traceback，状态标记为 `FAILED`，再抛出。
* `DAG._after_dag_end` 若找不到上下文会抛出错误，防止静默丢失。
* `check_serializable`：保障分布式/远端场景可运行。

---

## 9. 迷你示例

```python
with DAG("example_dag") as dag:
    a = SomeOp(node_name="A")
    b = SomeOp(node_name="B")
    c = SomeOp(node_name="C")
    a >> b >> c   # A -> B -> C

runner = DefaultWorkflowRunner()
dag_ctx = await runner.execute_workflow(c, call_data={"B": {"k": "v"}})

# 获取节点输出
out_b = dag_ctx.get_task_output("B")
```

* 在算子 `SomeOp._run` 中：

  1. 读取输入 `dag_ctx.current_task_context.task_input`；
  2. 处理逻辑；
  3. `set_task_output(...)` 写回结果；
  4. 可写入 `dag_ctx.save_task_share_data("B","key",data)`。

---

## 10. 要点总结

* **依赖关系**：`A >> B` 表示 A 在 B 上游。
* **唯一真相**：运行时结果都在 `DAGContext`。
* **执行顺序**：递归先上游再自己。
* **分支**：通过 metadata 标记下游跳过。
* **变量系统**：支持合并、覆盖、懒初始化。
* **并发**：目前串行，保留扩展点。
* **可观测性**：日志索引 + 链路追踪。
* **健壮性**：序列化检查、错误显式抛出。