# graphiti-memory-skill

一个 **Agent Skill**,教 AI 智能体正确使用 [Graphiti](https://github.com/getzep/graphiti)
MCP 服务器 —— 一种能跨会话存续的**知识图谱记忆**。

Graphiti 走的是开放的 [Model Context Protocol(MCP)](https://modelcontextprotocol.io),所以它
**不是 Claude 专属**:任何支持 MCP 的智能体 —— Claude Code、Claude Desktop、Cursor、Cline,或你
自己写的智能体 —— 都能接它,本 skill 的指引对它们都适用。它打包成 Claude Code 插件/skill 以便
一条命令安装,但 `SKILL.md` 是任何智能体都能加载的普通 Markdown。

Graphiti 不是扁平的键值存储。它先摄入**源情节(episode)**,再在后台从中抽取**实体**与**关系
事实**。这种图结构让它的工具很容易被误用 —— 搜错了分组、把排序靠前的猜测当真相,或者写入还在
排队就报"已保存"。本 skill 把正确的工作流固化下来,让智能体安全地读、写、验证和删除记忆。

## 它涵盖什么

- **心智模型** —— 情节 → 实体(节点)→ 事实(边),以及读取每一层各用哪个工具。
- **两条不可妥协的规则** —— 每次都钉死 `group_id`;没验证过就别说成功。
- **读取流程** —— 事实优先的检索、实体发现、出处核对,以及为何"空"≠"没有"。
- **写入流程** —— 自洽的情节、异步队列轮询、写后验证。
- **更正与删除** —— 外科手术式、需明确指令的删除;用新情节取代旧事实;`clear_graph` 的破坏性护栏。
- **完整工具参考** —— 全部 9 个 Graphiti 工具的每个参数、默认值和别名,见
  [`references/tools.md`](skills/graphiti-memory/references/tools.md)。

## 前置条件

一个正在运行、并已接入你的智能体 / MCP 客户端(Claude Code、Claude Desktop、Cursor、Cline,或
任何支持 MCP 的智能体)的 **Graphiti MCP 服务器**,暴露标准 Graphiti 工具(`add_memory`、
`search_memory_facts`、`search_nodes`、`get_episodes`、`get_entity_edge`、
`get_memory_queue_status`、`delete_entity_edge`、`delete_episode`、`clear_graph`)。参见
[Graphiti MCP 服务器文档](https://github.com/getzep/graphiti/tree/main/mcp_server)。

skill 本身与具体提供方、具体智能体无关 —— 它教的是*如何驱动这些工具*,不硬编码任何服务器 URL、
分组 ID 或凭据。

## 安装

方式 A、B 走 **Claude Code** 的插件/skill 加载器;方式 C 适用于任何其它智能体。

### 方式 A —— 作为 Claude Code 插件(推荐)

本仓库同时是一个自包含的插件市场(marketplace):

```
/plugin marketplace add batqwq/graphiti-memory-skill
/plugin install graphiti-memory@graphiti-memory-skill
```

### 方式 B —— 作为个人 skill,手动复制(Claude Code)

把 skill 文件夹复制进你的用户 skill 目录:

```bash
git clone https://github.com/batqwq/graphiti-memory-skill.git
cp -r graphiti-memory-skill/skills/graphiti-memory ~/.claude/skills/
```

然后重启 Claude Code(或重新加载 skill)。当任务涉及记住、回忆或管理 Graphiti 记忆时,skill 会
自动触发 —— 你不用手动调用它。

### 方式 C —— 任何其它支持 MCP 的智能体

skill 就是可移植的 Markdown,所以按你的智能体加载指令的方式喂给它即可 —— 系统提示词、规则文件
(如 Cursor 或 Cline 的 rule)、检索到的文档,或你框架自带的 skill 机制:

```bash
git clone https://github.com/batqwq/graphiti-memory-skill.git
# 把智能体指向:
#   skills/graphiti-memory/SKILL.md            (工作流 + 两条规则)
#   skills/graphiti-memory/references/tools.md (完整工具参数参考)
```

工作流和工具参考与智能体无关;只有安装这层"胶水"不同。

## 仓库结构

```
graphiti-memory-skill/
├── README.md
├── LICENSE
├── .claude-plugin/
│   ├── plugin.json            # 插件清单(Claude Code)
│   └── marketplace.json       # 自包含市场(Claude Code)
└── skills/
    └── graphiti-memory/
        ├── SKILL.md           # skill 本体(工作流 + 规则)—— 与智能体无关
        ├── references/
        │   └── tools.md       # 完整的逐工具参数参考
        └── evals/
            └── evals.json     # skill 的示例测试 prompt
```

## 工作原理

在 Claude Code 里,skill 分三级加载(渐进式披露):`name` + `description` 始终在上下文里;
`SKILL.md` 在 skill 触发时加载;`references/tools.md` 仅在需要确切参数时才拉进来。这让常驻开销很小,
同时把深层细节按需提供。其它智能体可以照搬这种分层,也可以直接加载 `SKILL.md` —— 内容不依赖任何
Claude 专属特性。

## 许可证

[MIT](LICENSE) © 2026 batqwq

Graphiti 是其各自作者的商标;本项目是独立的、社区编写的 skill,与 Zep / Graphiti 无隶属关系。
