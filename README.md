# 可插拔知识组件

从空库启动，导入自己的资料，给不同大模型共享同一份知识和经验。需要 Python 3.10 或更新版本，使用内置 SQLite。

## 导入与试用

在本目录运行。把示例路径换成自己的文件或目录。

```powershell
python knowledge.py --import "D:/资料/项目笔记" "D:/资料/验收规则.md"
'{"query":"验收规则"}' | python knowledge.py --call search_knowledge
'{}' | python knowledge.py --call knowledge_status
```

PowerShell 5 若传入中文 JSON，先运行 `$OutputEncoding = [System.Text.UTF8Encoding]::new($false)`。

支持 UTF-8 的 Markdown、TXT、JSON、JSONL、CSV 和 TSV，按文本切片。目录会递归导入。同一路径再次成功导入时更新已有文档，不增加副本。PDF、Word 和网页可以先导出为文本再导入。检索使用中文字符与双字词、英文词做关键词匹配，查询时优先使用资料里的术语。

## 配置到模型客户端

把以下条目合并到支持 MCP stdio 的客户端配置中。替换脚本绝对路径，必要时将 `python` 替换成 Python 可执行文件的绝对路径。

```json
{
  "mcpServers": {
    "portable-knowledge": {
      "command": "python",
      "args": ["D:/WorkSpace/AI/cqKnowledge/knowledge.py"]
    }
  }
}
```

重启客户端后，可以直接问“查一下项目的验收规则，并给我出处”。如果客户端没有把 MCP 初始化指引传给模型，可以将 `python knowledge.py --list-tools` 输出的 `instructions` 放进客户端指令。自建应用可通过 `--list-tools` 获取工具定义，通过 `--call 工具名` 以 UTF-8 JSON 输入参数和读取结果。也可以由应用先检索、再把结果传给模型。

## 让积累跨模型生效

- **沉淀经验**。明确告诉模型“记住这条经验”，并给出适用条件和验证依据。后续检索可同时找到导入资料和已保存的经验。
- **纠正检索**。告诉模型这次查询的某条来源相关或不相关。组件按原查询和来源记录反馈。相同或关键词高度重合的查询会调整该来源的分数，幅度最多为原分数的 20%。用 `reset` 撤销指定查询和来源的调整。
- **换模型或机器**。不同客户端使用同一个 SQLite 文件即可共享知识、经验和反馈。默认文件为脚本旁的 `.knowledge.sqlite3`。可以在导入命令和 MCP 参数中使用 `--db "数据库绝对路径"`。迁移时停止使用该库的客户端，再复制数据库文件。

组件在本地存储和检索，使用云端模型时，客户端可能会把检索片段作为上下文发给模型。

## 让模型持续整理知识

可以在 MCP 客户端里交代：“以后用这个知识库时，遇到有出处的新结论或验证过的变更，就帮我维护现有条目。碰到无法判定的冲突，告诉我有哪些依据。”客户端需要加载初始化指引，或把 `--list-tools` 返回的指引放进客户端配置。导入文件只会更新资料，整理动作由使用知识库的模型触发。

1. 模型先用 `search_knowledge` 找同主题的条目和原始依据，再用 `curate_knowledge` 提交标题、适用条件与结论、修改理由以及原文引用。每条引用使用检索结果中的 `document_id`、`chunk` 和逐字匹配的 `quote`。
2. 修订时提交原条目标题和当前 `expected_version`；如果另一客户端已更新，重新检索后再决定怎么修订。同一标题忽略大小写、全半角及多余空白。语义相近的主题由模型通过检索判断是否应当合并。
3. 普通检索只返回整理条目的当前版本，并附上原文引用。原始资料仍可独立检索。用 `knowledge_history` 可查看条目的旧结论、修订理由、保存时间和当时引用的摘录。

这一流程参考了 [Letta](https://github.com/letta-ai/letta) 的模型维护记忆、[Graphiti](https://github.com/getzep/graphiti) 的当前与历史知识，以及 [Mem0](https://github.com/mem0ai/mem0) 的变更记录。
