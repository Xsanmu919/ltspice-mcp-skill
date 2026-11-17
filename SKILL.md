---
name: ltspice-mcp
description: Connect a local LTspice 26 installation to this agent through LTspice's built-in MCP server, so the agent can open schematics, run SPICE simulations and read .raw waveforms. Use when the user asks to hook LTspice into Cursor, Claude Code, VS Code or DeepSeek Harness, or reports that LTspice tools are missing.
---

# LTspice MCP

LTspice 26.1.1+ ships an MCP server: `ltspice-mcp-bridge.exe` in the LTspice
install directory. The bridge speaks MCP over stdio and gives the agent 18 tools
for opening designs, editing schematics, running simulations and reading
waveforms. Nothing needs to be installed beyond LTspice itself.

**A skill does not create MCP tools by itself.** The tools appear only after the
host application is configured to launch the bridge and is restarted. Your job
here is to write that configuration, verify it, and explain the restart.

## 1. Preflight

Find the bridge. It lives next to `LTspice.exe`:

```
%LOCALAPPDATA%\Programs\ADI\LTspice\ltspice-mcp-bridge.exe
```

Check these in order and stop at the first hit:

- `$env:LOCALAPPDATA\Programs\ADI\LTspice\ltspice-mcp-bridge.exe`
- `$env:ProgramFiles\ADI\LTspice\ltspice-mcp-bridge.exe`
- `${env:ProgramFiles(x86)}\ADI\LTspice\ltspice-mcp-bridge.exe`
- `$env:ProgramFiles\LTC\LTspiceXVII\ltspice-mcp-bridge.exe`

If none exists, stop and report: the installed LTspice predates 26.1.1 and has
no MCP server. Do not try to work around it.

Windows only. The bridge is a Windows executable and LTspice does not ship a
Linux or macOS build.

## 2. Pick the target

| Host | Config file | Top-level key |
|---|---|---|
| Cursor (this project) | `.cursor/mcp.json` | `mcpServers` |
| Cursor (all projects) | `~/.cursor/mcp.json` | `mcpServers` |
| Claude Code (this project) | `.mcp.json` | `mcpServers` |
| Claude Code (all projects) | `claude mcp add --scope user` | — |
| VS Code / Copilot | `.vscode/mcp.json` | `servers` |
| DeepSeek Harness | a bundle, see `references/deepseek-harness.md` | — |

Ask which host the user is in if it is not obvious. Do not write more than one
unless asked.

## 3. Write the config

Preferred: run the bundled script, which resolves the bridge, merges into any
existing JSON and keeps a backup.

```powershell
# from this skill directory
powershell -NoProfile -ExecutionPolicy Bypass -File scripts/install-mcp.ps1 -Target cursor
```

Targets: `cursor`, `cursor-user`, `claude`, `vscode`. Add `-DryRun` to print the
result without touching disk. Add `-BridgePath "D:\LTspice\ltspice-mcp-bridge.exe"`
when the bridge is not in a default location.

Manual fallback — the entry is always this shape:

```json
{
  "mcpServers": {
    "ltspice": {
      "command": "C:\\Program Files\\ADI\\LTspice\\ltspice-mcp-bridge.exe",
      "args": [],
      "env": { "LTSPICE_INSTALL_DIR": "C:\\Program Files\\ADI\\LTspice" }
    }
  }
}
```

VS Code names the top-level key `servers` and accepts `"type": "stdio"`.
Claude Code at user scope is better done with the CLI:
`claude mcp add --scope user ltspice -- "<bridge path>"`.

Two rules when merging:

- Never replace the whole file. Read it, add or update the single `ltspice`
  entry, write it back.
- Keep the existing keys and other servers untouched; back up before writing.

## 4. Verify

This is the part that actually proves it works, and the user must do it:

1. Restart the host application (Cursor: reload window; Claude Code: new
   session).
2. Ask the agent to call a tool that needs no design open — `status`.
3. A healthy answer looks like:

```json
{ "current": { "backendPid": 12345, "committed": false, "state": "gui-attached" } }
```

If `status` is not in the tool list, the config did not load: check the path,
the JSON syntax, and the host's MCP log panel before changing anything else.

## 5. What the user gets

18 tools, in four groups:

- **Design** — `open_design`, `list_open_designs`, `get_active_design_path`,
  `get_design_content`, `set_design_content`, `get_schematic_netlist`,
  `start_simulation`, `is_simulation_running`, `get_spice_log`, `bring_to_front`
- **Waveforms** — `get_raw_info`, `read_raw_waveforms`, `export_raw_waveforms`
- **Docs** — `list_ltspice_reference_docs`, `get_ltspice_reference`
- **Session** — `status`, `attach`, `start_headless`

The bridge picks its mode automatically: it attaches to a running LTspice GUI if
one exists, otherwise it starts a hidden instance. The first state-changing call
commits the session to that instance.

## 6. Working rules worth passing on

- Before editing any `.asc`, read the schematic format reference
  (`get_ltspice_reference` with `SCHEMATIC-REFERENCE.md`). Do not hand-write
  `.asc` coordinates from memory.
- After any schematic edit, call `get_schematic_netlist` and check the netlist,
  not the drawing — a schematic can look right and be wired wrong.
- `.raw` files are large: `get_raw_info` first, then `read_raw_waveforms` with an
  explicit signal list and window, or `export_raw_waveforms` to SQLite.

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `status` missing from tool list | config not loaded, or wrong path | re-check JSON, restart host |
| Bridge starts then exits | path has spaces and lost its quoting | quote the `command` value |
| `failed to open document` | the `.asc` itself is malformed | open it in LTspice once |
| Tools work, no plots in the GUI | MCP runs the analysis without creating a plot window | press Run in LTspice, or read the `.raw` |

Platform details: `references/cursor.md`, `references/claude-code.md`,
`references/vscode.md`, `references/deepseek-harness.md`.
