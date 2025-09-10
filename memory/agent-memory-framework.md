# Agent Memory 系统解读

参考人类记忆的三阶段模型（感官记忆 → 短期记忆 → 长期记忆），代码实现了一个多层次的记忆框架。

---

## 一、设计思想：模拟人类记忆流程

记忆分为三层：

1. **感官记忆（SensoryMemory）**

   * 快速记录即时观测。
   * 超过阈值的片段才会进入短期记忆。
   * 模拟“看见/听见的信息，只留住有意义的部分”。

2. **短期记忆（ShortTermMemory / EnhancedShortTermMemory）**

   * 保存有限的近期记忆。
   * 有容量限制，触发溢出机制。
   * Enhanced 版引入 **embedding 相似度检索** 和 **增强计数**，可以捕捉重复/相关信息并进行提炼。

3. **长期记忆（LongTermMemory，接口预留）**

   * 通过 `transfer_to_long_term` 归档。
   * 搭配 **ImportanceScorer（重要性打分器）** 和 **InsightExtractor（洞察提取器）** 进行筛选和总结。

👉 总体流程：
**感官过滤 → 短期暂存 & 增强 → 长期归档 & 提炼洞察。**

---

## 二、关键抽象：MemoryFragment

最小存储单元，相当于“记忆碎片”。

**核心字段**：

* `raw_observation`：原始观测文本
* `embeddings`：语义向量（相似性计算）
* `importance`：重要性分数
* `last_accessed_time`：最近访问时间
* `is_insight`：是否为总结/洞察

**关键方法**：

* `calculate_current_embeddings()` → 生成向量
* `reduce(memory_fragments)` → 合并多个片段
* `update_importance()`、`update_embeddings()` → 更新属性

👉 作为 **最小存储粒度**，所有记忆层依赖它。

---

## 三、Memory 抽象接口

所有记忆类继承自 `Memory[T]`，统一定义：

* `write()`：写入记忆
* `read()`：检索记忆
* `clear()`：清空
* `transfer_to_long_term()`：转移长期记忆
* `get_insights()`：调用 LLM 提炼洞察
* `score_memory_importance()`：打分
* `handle_overflow()`：容量溢出策略
* `handle_duplicated()`：去重策略

👉 模块化设计，便于扩展。

---

## 四、感官记忆（SensoryMemory）

**关键成员**：

* `_buffer_size`：缓冲区大小
* `_fragments`：片段存储
* `importance_weight = 0.9`
* `threshold_to_short_term = 0.1`

**核心逻辑**：

* `write()` → 写入片段并检查溢出。
* `handle_overflow()`：

  * 打分 ≥ 阈值 → 转入短期记忆
  * 否则丢弃
* `read()` → 返回当前缓冲区内容

👉 模拟 **瞬时感官存储 → 过滤 → 有价值内容进入短期记忆**。

---

## 五、短期记忆（ShortTermMemory）

**关键成员**：

* `_buffer_size`
* `_fragments`
* `_lock`（并发安全）

**核心逻辑**：

* `write()`：

  1. 去重
  2. 写入缓冲区
  3. 若超容量 → `transfer_to_long_term()`
  4. 调用 `handle_overflow()`
* `transfer_to_long_term()`：超容量时丢弃最旧片段
* `read()`：返回 `_fragments`

👉 模拟 **有限容量（7±2） → 新进旧出 → 部分归档**。

---

## 六、增强短期记忆（EnhancedShortTermMemory）

**新增机制**：相似性增强。

**关键成员**：

* `short_embeddings`：片段向量
* `enhance_cnt`：增强计数
* `enhance_memories`：相似片段集合
* `enhance_similarity_threshold = 0.7`
* `enhance_threshold`：计数超过该值 → 合并归档

**核心逻辑**：

1. **write()**

   * 计算 embedding → 与已有片段算相似度
   * 若超过阈值 → `enhance_cnt +1` 并存入 `enhance_memories`
   * 调用 `handle_overflow()`

2. **transfer\_to\_long\_term()**

   * 若 `enhance_cnt >= enhance_threshold`
   * 合并片段 (`reduce()`) → 提炼洞察 → 存入长期记忆
   * 清理对应缓存

3. **handle\_overflow()**

   * 超容量时 → 按 `(importance, enhance_cnt)` 排序
   * 丢弃最不重要、最少增强的片段

👉 模拟 **人类对重复/相关信息的强化记忆 → 合并总结 → 归档**。

---

## 七、相关性检索机制

在 **EnhancedShortTermMemory** 中：

1. 新片段生成 embedding
2. 与已有片段做 **cosine similarity**
3. 用 `sigmoid` 映射到 \[0,1]
4. 概率 ≥ 阈值（0.7） → 判定为相关
5. 增强计数 +1，并存入 `enhance_memories`

👉 **概率性增强**（非确定性），模拟人类“偶然联想/触发”的机制。

---

## 八、总结（层层递进）

1. **SensoryMemory** → 过滤输入，保留高重要性片段
2. **ShortTermMemory** → 有限容量，新进旧出，部分转长期
3. **EnhancedShortTermMemory** → 相似性增强，触发总结/归档
4. **LongTermMemory** → 存储提炼洞察和高价值记忆

👉 整体系统模拟了 **遗忘机制 + 联想增强 + 总结归档**。
