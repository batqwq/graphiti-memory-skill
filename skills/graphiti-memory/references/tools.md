# Graphiti MCP — 工具参考

每个 Graphiti MCP 工具的确切参数、默认值与行为。实际工具名由 MCP 服务器加命名空间前缀(通常是
`mcp__<server>__<tool>`,例如 `mcp__Graphiti__add_memory`);下面用的是服务器文档里的逻辑名。

先读 [`../SKILL.md`](../SKILL.md) 了解流程和两条不可妥协的规则。本文件是查表用的。

## 目录

- [写入](#写入) — `add_memory`
- [读取](#读取) — `search_memory_facts`、`search_nodes`、`get_entity_edge`、`get_episodes`
- [状态](#状态) — `get_status`、`get_memory_queue_status`
- [删除](#删除) — `delete_entity_edge`、`delete_episode`、`clear_graph`

---

## 写入

### `add_memory`

把**一条**源情节排队,异步抽取进图。返回值只确认情节已**排队** —— 不代表处理完成。

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `name` | string | —(必填) | 该情节的简短、稳定、描述性标题。 |
| `episode_body` | string | —(必填) | 完整源内容。当 `source="json"` 时,传 JSON **编码后的字符串**,而非对象。 |
| `group_id` | string \| null | `null` | 已确认的确切分组。实践中必填 —— 不知道就问用户;不要省略或猜测。 |
| `source` | string | `"text"` | `"text"`(散文)、`"json"`(JSON 编码字符串)或 `"message"`(对话式)。 |
| `source_description` | string | `""` | 简短出处,如文档类型或对话上下文。 |
| `uuid` | string \| null | `null` | 可选的调用方自带稳定情节 UUID。通常省略。 |

**调用后:** 用相同的 `group_id` 轮询 [`get_memory_queue_status`](#get_memory_queue_status),直到
`pending=0` 且 `processing=0`,再用 [`search_memory_facts`](#search_memory_facts) 验证。写明确的
名称、关系、日期、出处和不确定性;一条情节只放一个事件;不用代词或关键词碎片。

---

## 读取

### `search_memory_facts`

主检索工具。按语义搜索派生出的关系**事实(边)**。用于事实、关系和时间相关的问题。

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `query` | string | —(必填) | 一句聚焦的自然语言疑问句,点名实体、关系和时间约束。不是关键词列表。 |
| `group_ids` | string[] \| null | `null` | 已确认的确切分组。实践中必填 —— 绝不扩大范围或跨未确认分组检索。 |
| `center_node_uuid` | string \| null | `null` | 可选的实体节点 UUID(来自 `search_nodes`),把检索聚焦到该实体周围。传真实 UUID,绝不传名称。 |
| `max_facts` | integer | `10` | 返回的最大排序事实数;正整数,受服务器上限约束。 |

结果是按相关性排序的**候选**,不是保证为真。重要的用 `get_entity_edge` 核实。

### `search_nodes`

按名称、别名、角色、摘要或语义发现实体**节点**。用于实体发现或获取 UUID —— 不用于关系问题
(那些用 `search_memory_facts`)。

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `query` | string | —(必填) | 聚焦的实体查找:规范名、别名、角色或可识别的上下文。 |
| `group_ids` | string[] \| null | `null` | 已确认的确切分组。实践中必填。 |
| `entity_types` | string[] \| null | `null` | 可选,按配置的确切实体标签过滤。不清楚就省略。 |
| `max_nodes` | integer | `10` | 返回的最大排序节点数;正整数,受服务器上限约束。 |

节点的 `summary` 是生成的上下文,可能过时 —— 关键论断要对照事实或情节核实。空结果不等于不存在。

### `get_entity_edge`

按 UUID 取回**一条**事实(边)的完整已存表示。只读。

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `uuid` | string | —(必填) | 来自 `search_memory_facts`、在已确认分组中的确切事实 UUID。绝不猜测。 |

用它在依赖或删除一条事实之前,检查其出处、时间字段(生效/失效)和属性。

### `get_episodes`

按 `created_at` 的确定顺序,列出一个或多个分组的原始**源情节**。出处 / 近期历史核对 ——
**不是**语义搜索。

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `group_ids` | string[] \| null | `null` | 已确认的确切分组。优先于旧版 `group_id`。 |
| `max_episodes` | integer \| null | `null` | 首选页大小;正整数,受服务器上限约束。 |
| `offset` | integer | `0` | 用于分页时跳过的情节数。 |
| `sort_order` | string | `"desc"` | `"desc"`(最新在前)或 `"asc"`(最旧在前)。 |
| `content_max_chars` | integer | `1200` | 每条情节的预览长度;`0` 表示不返回内容预览。 |
| `group_id` | string \| null | `null` | 旧版单分组别名。优先用 `group_ids`。 |
| `limit` / `last_n` | integer \| null | `null` | 旧版页大小别名。优先用 `max_episodes`;不要传互相矛盾的值。 |

用它在 `delete_episode` 之前定位并核实一个情节 UUID。

---

## 状态

### `get_status`

仅连通性诊断 —— 确认 MCP 服务已初始化、能访问其图数据库。无参数。

结果健康**不**代表队列已空、记忆存在,或 LLM / 嵌入提供方对新写入正常工作。摄入健康度请用
`get_memory_queue_status`。

### `get_memory_queue_status`

检查异步摄入进度,不修改图。

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `group_id` | string \| null | `null` | 要检查队列的分组。`add_memory` 之后,传**相同的** group_id。仅在做全局管理总览时省略。 |

只有当 `pending` **和** `processing` 都为 `0` 时,该分组才算追平。任何 `failed` 计数或
`last_error` 都要汇报;非零的 `retried` 表示瞬时 JSON 输出失败已自动恢复(诊断可靠性时有用)。

---

## 删除

> 删除是外科手术式的、需明确指令。只移除用户点名的那一条,且仅在他们明确要求时。绝不要因为结果
> 看着重复、过时、嘈杂或不对就删。见 [SKILL.md → 更正与删除](../SKILL.md#更正与删除)。

### `delete_entity_edge`

按 UUID 永久删除恰好一条派生**事实(边)**。

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `uuid` | string | —(必填) | 经 `search_memory_facts` 定位、`get_entity_edge` 核实的确切事实 UUID。绝不猜测。 |

不会删除源情节或相连的节点;残留的情节可能导致相似事实被重新派生。事后在同一分组重新搜索以确认。

### `delete_episode`

按 UUID 永久删除恰好一条源**情节**。

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `uuid` | string | —(必填) | 经 `get_episodes` 核实的确切情节 UUID。绝不猜测。 |

移除该源记录及其直接连接,但不保证清除它曾影响的每个节点、派生事实或生成的摘要。事后在同一分组核实。

### `clear_graph`

**不可逆。** 删除指定分组分区里的所有情节、节点和事实。不创建备份。

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `group_ids` | string[] \| null | `null` | 用户点名并授权永久抹除的每一个分组的显式列表。绝不省略、猜测或扩大范围。 |

只有当用户明确要求重置、点名了每一个目标分组、并确认可以永久删除时,才调用。它不是清理、去重、
迁移或排障工具。事后只核实被显式授权的分组,并汇报结果。
