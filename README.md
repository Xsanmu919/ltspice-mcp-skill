# LTspice MCP skill

One Agent Skill that connects a local **LTspice 26** installation to an AI agent —
Cursor, Claude Code, VS Code Copilot or DeepSeek Harness — through LTspice's own
MCP server, so the agent can open schematics, run SPICE simulations and read
`.raw` waveforms.

> 一个 Agent Skill：把本机 LTspice 26 自带的 MCP 服务端接进各种 AI agent，让它能开原理图、跑仿真、读波形。

## 先说清楚一件事

**Skill 本身不提供 MCP 工具。** Skill 是一份操作说明：它告诉 agent 去哪里找
`ltspice-mcp-bridge.exe`、把配置写到哪个文件、怎么验证。真正让工具出现的是
宿主应用读到你写进去的那份 MCP 配置并重启。

所以安装分两步：让 agent 用这个 skill 写好配置 → 重启宿主。

## 前置条件

| 项目 | 要求 |
|---|---|
| LTspice | **26.1.1 或更高**，安装目录里要有 `ltspice-mcp-bridge.exe` |
| 操作系统 | **Windows**（LTspice 本身只发布 Windows 版） |
| 宿主 | Cursor 2.4+ / Claude Code / VS Code / DeepSeek Harness |

旧版 LTspice XVII 没有这个 bridge，用不了。

## 安装 skill

Skill 就是「一个目录 + 一份 SKILL.md」。放到任意一个宿主能扫到的位置：

```powershell
# Claude Code 与 Cursor 都能读这个目录
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse -Force . "$env:USERPROFILE\.claude\skills\ltspice-mcp"
```

其它可用位置：`.cursor/skills/`、`.agents/skills/`、`.claude/skills/`（项目级），
以及 `~/.cursor/skills/`、`~/.agents/skills/`（用户级）。

## 安装 MCP 连接

配置这一步可以让 agent 代劳（它读完 SKILL.md 就知道该怎么做），也可以自己跑脚本：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File scripts/install-mcp.ps1 -Target cursor
```

| `-Target` | 写入文件 | 顶层键 |
|---|---|---|
| `cursor` | `.cursor/mcp.json` | `mcpServers` |
| `cursor-user` | `~/.cursor/mcp.json` | `mcpServers` |
| `claude` | `.mcp.json` | `mcpServers` |
| `vscode` | `.vscode/mcp.json` | `servers` |

脚本会：探测 bridge 路径 → 读取已有配置 → 只合并 `ltspice` 这一项 → 先备份再写回。
`-DryRun` 只打印不落盘。

然后**重启宿主**，让 agent 调一个 `status` 工具验收：

```json
{ "current": { "backendPid": 12345, "committed": false, "state": "gui-attached" } }
```

## 装好之后

18 个工具：

- **设计** `open_design`、`get_design_content`、`set_design_content`、`get_schematic_netlist`、`start_simulation`、`get_spice_log` …
- **波形** `get_raw_info`、`read_raw_waveforms`、`export_raw_waveforms`
- **文档** `list_ltspice_reference_docs`、`get_ltspice_reference`
- **会话** `status`、`attach`、`start_headless`

bridge 会自动选模式：你开着 LTspice 就挂到那个窗口（GUI-attached），否则起一个隐藏实例。

## 目录结构

```
SKILL.md                  技能主体：判断、定位、配置、验证、排错
references/
  cursor.md               Cursor 的配置位置与调试面板
  claude-code.md          Claude Code 的 mcp.json 与 skills 目录
  vscode.md               VS Code 的 servers 键名差异
  deepseek-harness.md     DSH 的 bundle 形式
scripts/
  install-mcp.ps1         探测 + 合并写配置（带备份、DryRun）
assets/
  icon.svg
```

## 相关

- LTspice 的 MCP 桥接在 Cursor 里的实测记录：<https://github.com/Xsanmu919/dsh-ltspice-mcp>
- Cursor Skills 文档：<https://cursor.com/docs/skills>
- Cursor MCP 文档：<https://cursor.com/help/customization/mcp>

## License

MIT
