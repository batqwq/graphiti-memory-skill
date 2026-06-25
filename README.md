# graphiti-memory-skill

一个 **Agent Skill**,教 AI 智能体正确使用
[Graphiti](https://github.com/getzep/graphiti) MCP 知识图谱记忆。

Graphiti 不是扁平键值存储 —— 它摄入**源情节**,后台抽取**实体**与**关系事实**。这种图结构让
工具容易被误用:搜错分组、把候选当真相、写入还在排队就报"已保存"。本 skill 把正确的工作流
固化下来,让智能体安全地读、写、验证和删除记忆。

## 涵盖内容

- **心智模型** — 情节 → 实体(节点)→ 事实(边),每层对应的工具
- **两条核心规则** — 钉死 `group_id`;没验证过不说成功
- **读取流程** — 事实优先检索、实体发现、出处核对
- **写入流程** — 自洽情节、队列轮询、写后验证
- **更正与删除** — 外科手术式删除、用新情节取代旧事实、`clear_graph` 护栏
- **工具参考** — 9 个工具的参数、默认值、别名,见 [`references/tools.md`](skills/graphiti-memory/references/tools.md)

## 前置条件

一个正在运行的 **Graphiti MCP 服务器**,已接入你的 MCP 客户端,暴露标准工具
(`add_memory`、`search_memory_facts`、`search_nodes`、`get_episodes` 等)。
参见 [Graphiti MCP 服务器文档](https://github.com/getzep/graphiti/tree/main/mcp_server)。

## 安装

### 作为 Claude Code 插件(推荐)

```
/plugin marketplace add batqwq/graphiti-memory-skill
/plugin install graphiti-memory@graphiti-memory-skill
```

### 手动复制

```bash
git clone https://github.com/batqwq/graphiti-memory-skill.git
cp -r graphiti-memory-skill/skills/graphiti-memory ~/.claude/skills/
```

### 其它 MCP 客户端

`SKILL.md` 和 `references/tools.md` 是普通 Markdown,按你的客户端加载指令的方式喂给它即可(系统提示词、规则文件等)。

## 仓库结构

```
graphiti-memory-skill/
├── README.md
├── LICENSE
├── .claude-plugin/           # Claude Code 插件清单
└── skills/
    └── graphiti-memory/
        ├── SKILL.md           # 工作流 + 规则
        ├── references/
        │   └── tools.md       # 逐工具参数参考
        └── evals/
            └── evals.json     # 示例测试 prompt
```

## 许可证

[MIT](skills/graphiti-memory/LICENSE.txt) © 2026 batqwq
