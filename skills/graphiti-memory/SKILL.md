---
name: graphiti-memory
description: "This skill should be used when the user asks to \"remember this\", \"save to memory\", \"recall what we discussed\", \"what do you know about X\", \"check memory for\", \"correct that memory\", \"delete that fact\", \"clear the graph\", or uses any Graphiti MCP tool (add_memory, search_memory_facts, search_nodes, get_episodes, get_entity_edge, get_memory_queue_status, delete_entity_edge, delete_episode, clear_graph). Also use when the user mentions group_id, knowledge graph memory, or cross-session persistence — even without saying \"Graphiti\". Graphiti is a graph (episodes to entities to facts) that behaves very differently from a key-value store; consult this skill before calling its tools."
version: 1.0.0
---

# Graphiti 记忆

Graphiti 是持久化知识图谱记忆服务。它摄入**源情节(episode)**,由 LLM 在后台抽取**实体(节点)**
与**关系事实(边)**。检索的是含义,不是原文。

## 三层结构

| 层 | 含义 | 读取工具 |
|----|------|----------|
| 情节 | 写入的原始素材(笔记、消息、JSON) | `get_episodes` |
| 节点(实体) | 抽取出的人/项目/地点/事物 | `search_nodes` |
| 边(事实) | 实体间的关系,带时间有效性 | `search_memory_facts` → `get_entity_edge` |

写入自上而下(添加情节 → 异步派生实体和事实);读取从事实层开始,需要时追溯出处。

## 两条核心规则

**1. 钉死 group_id。** 每个分组作用域的调用必须带已确认的确切 group_id/group_ids。分组是硬隔离。
不清楚时先问用户,绝不猜测、臆造、扩大范围或省略。

**2. 先验证再说成功。** 搜索结果是候选而非证据;写入只是排队;删除可能不完全传播。任何变更后,
跑后续核对并汇报实际观察到的结果。

## 选对工具

```
事实/关系/时间问题           → search_memory_facts
发现实体或拿 UUID            → search_nodes
查看一条事实的完整细节       → get_entity_edge
出处/原始历史                → get_episodes
记住某件事                   → add_memory
写入是否完成                 → get_memory_queue_status
服务是否在线                 → get_status
删除一条事实/情节            → delete_entity_edge / delete_episode
清空整个分组                 → clear_graph（见破坏性操作）
```

完整参数表见 [`references/tools.md`](references/tools.md)。

## 读取流程

1. 对事实/关系/时间问题,先用 `search_memory_facts`。写一句聚焦的自然语言疑问句,点名实体、关系和
   时间约束。不要用关键词堆砌——查询作为整体被向量化。
2. 仅在需要发现实体或拿 UUID 时用 `search_nodes`。节点摘要是生成的上下文,非权威证据。
3. 用 `get_entity_edge` 核实重要事实的出处和时间字段。
4. 用 `get_episodes` 做出处核对或回顾原始历史。
5. 空结果不等于不存在——换更具体的查询、核对 group_id、检查 `get_memory_queue_status`。

## 写入流程

1. **先查重。** 用 `search_memory_facts` 在同一分组搜索有没有相同或高度相似的已存事实。已存且准确
   则告知用户"已记着了";已存但过时则走更正流程。仅确认无相似记录时才写入。
2. **一次 `add_memory` 写一条自洽的情节。** 写明确的实体名称(不用代词)、关系、日期、出处和
   不确定性。多个不相关事件拆开。
3. **只存用户真正想记住的内容。** 不要悄悄持久化猜测或工作草稿。
4. **轮询队列。** 用相同 group_id 调用 `get_memory_queue_status`,直到 pending=0 且
   processing=0。非零 failed 或 last_error 必须汇报。
5. **验证可检索。** 队列清空后用 `search_memory_facts` 确认事实能检索到,再告知用户。

## 更正与删除

优先**添加新的准确情节**来取代旧事实(Graphiti 双时态,新事实自动取代旧的),而非删除。

删除仅用于用户明确要求移除的那一条:
1. 用 `search_memory_facts` 定位,`get_entity_edge` 核实 UUID。绝不猜 UUID。
2. 调用 `delete_entity_edge`(事实)或 `delete_episode`(情节)。
3. 事后在同一分组重新搜索并汇报核实结果。

### 破坏性操作：clear_graph

不可逆地抹掉指定分组的全部数据,无备份。仅当用户明确要求重置、点名每个目标分组、且确认永久删除时
才调用。三者缺一则停下来问。绝不用它来"修复"嘈杂/重复的结果。

## 常见陷阱

- 不查重就写入 → 重复情节产出重复事实
- 省略 group_id → 结果来自错误分组
- 搜索第一名当真相 → 应先 `get_entity_edge` 核实
- add_memory 后直接报"存好了" → 只是排队,先轮询再验证
- 关键词堆砌查询 → 写成疑问句
- 节点摘要当事实引用 → 摘要是生成的,事实边才是证据
- 为"清理"而删除 → 删除只用于明确点名的移除

## 参考文件

- **[`references/tools.md`](references/tools.md)** — 全部 9 个工具的参数、默认值、别名和调用注意事项
