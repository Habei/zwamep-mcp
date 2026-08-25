[README.md](https://github.com/user-attachments/files/31400424/README.md)
# ZwAmepMcpServer · 中望 AMEP MCP 服务

> Bridge AI agents to ZWCAD 2027 AMEP (architecture & HVAC) — 110 drafting tools over stdio MCP.

[![Platform](https://img.shields.io/badge/Windows-10%2F11%20x64-0078d4)](#系统要求) [![CAD](https://img.shields.io/badge/ZWCAD-2027%20HVAC-orange)](#系统要求) [![MCP](https://img.shields.io/badge/MCP-stdio-blueviolet)](#架构) [![Tools](https://img.shields.io/badge/Tools-110-success)](#工具一览)

[English](#english) · [中文](#中文)

---

## English

**ZwAmepMcpServer** is a single-file MCP server (PyInstaller distribution) that connects AI clients (Claude Desktop, Claude Code, Cursor, etc.) to **ZWAMEP 2027 Forever** for architectural & mechanical-electrical-plumbing (AMEP) drafting.

It exposes **110 tools** through the Model Context Protocol:

| Prefix     | Count | Transport  | Description                                                      |
| ---------- | ----: | ---------- | ---------------------------------------------------------------- |
| `amep_*`   |    57 | Named pipe | Architecture & HVAC: walls, openings, columns, ducts, valves, fans, etc. |
| `com_*`    |    53 | COM        | Generic drafting: draw, edit, dimension, blocks, text, layers, etc. |

### How it works

```
┌──────────────┐   stdio (MCP)   ┌────────────────────┐  named pipe  ┌────────────────────┐
│  AI client   │ ──────────────► │  ZwAmepMcpServer   │ ───────────► │  ZWCAD 2027 HVAC  │
│  (Claude…)   │ ◄────────────── │       (exe)        │ ◄─────────── │  + 2 ZRX plugins  │
└──────────────┘   JSON-RPC      └────────────────────┘  amep_*       └────────────────────┘
                                                                              +
                                                                          com_* (COM)
```

The exe is only the **front-end**. The actual CAD capability lives in two companion ZRX plugins loaded inside the ZWCAD process:

- `ZAecAMEPServerBridge.zrx` — exposes the `amep_*` tools over the named pipe `\\.\pipe\amep_server_bridge_pipe`.
- `ZAecAMEPServerCom.zrx` — exposes the `com_*` tools via COM.

### Requirements

- Windows 10/11 x64
- ZWAMEP2027、ZWAMEP2027 365
- Any MCP client supporting stdio transport
- Python is **not** required (already bundled in the exe)

### Install

1. **Unpack** the distribution to a fixed location, e.g. `C:\AMEP_MCP\`.
2. **Load the ZRX plugins in ZWCAD**:
   - Run `APPLOAD` → load `zrx\ZAecAMEPServerBridge.zrx`. The command line should print
     `[ZAecAMEPServerBridge] 桥接插件已加载，命名管道就绪。MCP Server 可连接。`
   - Load `zrx\ZAecAMEPServerCom.zrx` as well.
   - *(Recommended)* Add both to the **Startup Suite** so they auto-load on ZWCAD launch.
   - Verify: run `AMEPSERVERBRIDGE` → it should report the pipe as `运行中`.
3. **Configure your MCP client** (`claude_desktop_config.json` or equivalent):

   ```json
   {
     "mcpServers": {
       "zwamep": {
         "command": "C:\\AMEP_MCP\\ZwAmepMcpServer.exe",
         "args": ["--mode", "named_pipe"]
       }
     }
   }
   ```

   Claude Code equivalent:
   ```bash
   claude mcp add amep -- "C:\AMEP_MCP\ZwAmepMcpServer.exe" --mode named_pipe
   ```

### Verify

1. Open a drawing in ZWCAD (e.g. `Drawing1.dwg`).
2. From the AI client, call `amep_get_app_info` → expect ZWCAD version + current document.
3. Call `com_get_cad_info` → expect `cad_name: ZWHVAC` and `com_version: ZAecAMEPServerCom 1.0`.
4. Smoke test: `com_draw_radiator(x=0, y=0)` → a radiator should appear in ZWCAD.

### Optional config — `config\amep_mcp_config.json`

Read at startup; falls back to built-in defaults if missing.

```json
{
  "bridge": {
    "mode": "named_pipe",
    "pipe_name": "amep_server_bridge_pipe",
    "pipe_timeout_secs": 10
  }
}
```

- `mode`: `named_pipe` (normal use) or `mock` (no-CAD regression self-test, all tools return fake data).
- `pipe_name`: must match what `ZAecAMEPServerBridge.zrx` was compiled with (default `amep_server_bridge_pipe`); usually do not change.

### Troubleshooting

| Symptom | Cause / fix |
|---|---|
| MCP client can't connect to exe | Wrong `command` path; or AV / SmartScreen blocking (exe is unsigned) — add to allowlist and retry. |
| All `amep_*` tools time out / pipe error | ZWCAD didn't load `ZAecAMEPServerBridge.zrx`, or a version mismatch. Run `AMEPSERVERBRIDGE` in ZWCAD to confirm the pipe is running. |
| `com_*` tools fail with "ProgID by ZAecAMEPServerCom.zrx" | ZWCAD didn't load `ZAecAMEPServerCom.zrx`. |
| `com_*` works but `amep_*` reports `loadModule` failure | `ZAecArchService.zrx` / `ZAecHvacService.zrx` and their DLLs are not in the search path — copy the whole `zrx\` directory. |
| Pipe call succeeds but no change in drawing | Make sure a drawing is open; some tools require a host entity first (e.g. openings need a wall). |
| `APPLOAD` reports missing DLLs | Companion DLLs are not shipped — copy them from the dev machine's `Out\VC15\Alpha_HVAC\x64\Bin\HVAC27\`. |
| Version mismatch / no response after load | zrx and ZWCAD versions must match (currently 2027 / HVAC27); cross-version is not supported. |

### Notes

- **Single ZWCAD instance**: COM is bound to the running ZWCAD process. When multiple are open, the one with the registered ProgID is connected — keep only one running.
- **Recompiling zrx**: unload the old build in ZWCAD and load the new one (or restart ZWCAD). After the pipe disconnects, MCP tools auto-reconnect.
- **Version coupling**: the wire protocol inside the exe matches the pipe protocol / method names compiled into the zrx — upgrade the whole package together, never one side.

---

## 中文

**ZwAmepMcpServer** 是一款单文件 MCP 服务（PyInstaller 打包），把 AI 客户端（Claude Desktop / Claude Code / Cursor 等）连接到 **ZWAMEP 2027永久版、ZWAMEP 2027 365版本**，用于建筑、水暖电（AMEP）专业出图。

通过 Model Context Protocol 暴露 **110 个工具**：

| 前缀        | 数量 | 传输方式   | 说明                                        |
|------------|----:|-----------|---------------------------------------------|
| `amep_*`   |  57 | 命名管道   | 建筑构件 / 风管 / 风阀 / 风机 等              |
| `com_*`    |  53 | COM       | 通用绘图 / 编辑 / 标注 / 块 / 文字 / 图层 等   |

### 架构

```
┌──────────────┐   stdio (MCP)   ┌────────────────────┐  命名管道    ┌────────────────────┐
│  AI 客户端    │ ──────────────► │  ZwAmepMcpServer   │ ───────────► │  ZWCAD 2027 暖通   │
│  (Claude…)   │ ◄────────────── │     （exe）        │ ◄─────────── │  + 2 个 ZRX 插件   │
└──────────────┘   JSON-RPC      └────────────────────┘  amep_*      └────────────────────┘
                                                                              +
                                                                          com_* (COM)
```

exe 只是个**前端**。真正的画图能力在 ZWCAD 进程内的两个 ZRX 插件里：

- `ZAecAMEPServerBridge.zrx` — 通过命名管道 `\\.\pipe\amep_server_bridge_pipe` 提供 `amep_*` 工具。
- `ZAecAMEPServerCom.zrx` — 通过 COM 提供 `com_*` 工具。

### 系统要求

- Windows 10/11 x64
- ZWAMEP 2027 永久版、ZWAMEP 2027 365版本
- 任意支持 stdio 传输的 MCP 客户端
- **无需** Python（已打包进 exe）

### 安装

1. **解压**到固定目录，例如 `C:\AMEP_MCP\`。
2. **在 ZWCAD 中加载 ZRX 插件**：
   - 命令行执行 `APPLOAD` → 加载 `zrx\ZAecAMEPServerBridge.zrx`，命令行应出现
     `[ZAecAMEPServerBridge] 桥接插件已加载，命名管道就绪。MCP Server 可连接。`
   - 同时加载 `zrx\ZAecAMEPServerCom.zrx`。
   - *（推荐）* 把两个 zrx 加入「启动套件（Startup Suite）」，下次启动 ZWCAD 自动加载。
   - 验证：执行 `AMEPSERVERBRIDGE` → 应输出管道 `运行中`。
3. **配置 MCP 客户端**（以 `claude_desktop_config.json` 为例）：

   ```json
   {
     "mcpServers": {
       "zwamep": {
         "command": "C:\\AMEP_MCP\\ZwAmepMcpServer.exe",
         "args": ["--mode", "named_pipe"]
       }
     }
   }
   ```

   Claude Code 等价命令：
   ```bash
   claude mcp add amep -- "C:\AMEP_MCP\ZwAmepMcpServer.exe" --mode named_pipe
   ```

### 验证

1. 在 ZWCAD 中打开一张图纸（如 `Drawing1.dwg`）。
2. 在 AI 客户端调用 `amep_get_app_info` → 应返回 ZWCAD 版本与当前文档。
3. 调用 `com_get_cad_info` → 应返回 `cad_name: ZWHVAC`、`com_version: ZAecAMEPServerCom 1.0`。
4. 跑通最小测试：`com_draw_radiator(x=0, y=0)` → ZWCAD 中出现散热器即全链路通。

### 可选配置 — `config\amep_mcp_config.json`

exe 启动时读取；缺省走内置默认。

```json
{
  "bridge": {
    "mode": "named_pipe",
    "pipe_name": "amep_server_bridge_pipe",
    "pipe_timeout_secs": 10
  }
}
```

- `mode`：`named_pipe`（正常使用）/ `mock`（无 CAD 环境回归自测，所有工具返回假数据）。
- `pipe_name`：须与 `ZAecAMEPServerBridge.zrx` 编译期一致（默认 `amep_server_bridge_pipe`），一般不改。

### 故障排查

| 现象 | 原因与处理 |
|---|---|
| MCP 客户端连不上 exe | `command` 路径写错；或被杀软 / SmartScreen 拦截（exe 未签名），加白名单后重试。 |
| 所有 `amep_*` 工具超时 / pipe 错误 | ZWCAD 未加载 `ZAecAMEPServerBridge.zrx`，或加载的不是同版本编译产物；执行 `AMEPSERVERBRIDGE` 确认管道运行中。 |
| `com_*` 工具报「该 ProgID 由 ZAecAMEPServerCom.zrx 加载」 | ZWCAD 未加载 `ZAecAMEPServerCom.zrx`。 |
| `com_*` 可用但 `amep_*` 报 `loadModule` 失败 | `ZAecArchService.zrx` / `ZAecHvacService.zrx` 及其依赖 dll 不在搜索路径；整目录拷贝。 |
| 管道工具调了但图纸无变化 | 确认 ZWCAD 有打开的图纸；部分工具需先有宿主实体（如门窗需先有墙）。 |
| `APPLOAD` 报缺 dll | 依赖 dll 未随包分发；从开发机 `Out\VC15\Alpha\x64\Bin\Arch27\` 整目录拷贝。 |
| 版本不匹配 / 加载后无反应 | zrx 与 ZWCAD 版本必须配套（当前 2027），跨版本不通用。 |

### 注意事项

- **单 ZWCAD 实例**：COM 连接绑定到正在运行的 ZWCAD 进程，多开时连的是先启动（已注册 ProgID）的那个，建议只开一个。
- **zrx 重新编译**：需在 ZWCAD 里卸载旧版再加载新版（或重启 ZWCAD），管道断开后 MCP 工具会自动重连。
- **版本配套**：exe 内置协议与 zrx 的管道协议 / 方法名是配套版本，整包一起升级，不要单独替换其一。

---

## 工具一览

### `amep_*` — 命名管道工具（57 个）

| 类别 | 代表工具 | 说明 |
|---|---|---|
| 应用/文档 | `amep_get_app_info`、`amep_doc_list`、`amep_doc_open`、`amep_doc_save` | 应用信息、文档管理 |
| 墙体 | `amep_arch_create_wall`、`amep_arch_create_walls`、`amep_arch_align_column_to_wall` | 直墙、连续折线墙、柱齐墙 |
| 柱 | `amep_arch_create_column` | 矩形/圆形/正多边形截面柱 |
| 门窗/洞口 | `amep_arch_create_opening`、`amep_arch_create_band_window`、`amep_arch_create_corner_window` | 普通洞口、带形窗、角窗 |
| 楼板/散水 | `amep_arch_create_apron` | 散水 + 平台 |
| 房间/面积 | `amep_arch_search_space`、`amep_arch_query_area` | 由墙搜索房间、面积查询 |
| 楼梯 | `amep_arch_create_auto_stair`、`amep_arch_create_rect_stair`、`amep_arch_create_scissors_stair` | 自动/矩形/剪刀楼梯 |
| 电梯/阳台 | `amep_arch_create_elevator`、`amep_arch_create_balcony` | 电梯、阳台 |
| 洁具/隔断 | `amep_arch_create_sanitary`、`amep_arch_create_partition_room` | 卫生洁具、隔断 |
| 风管 | `amep_hvac_create_straight_duct`、`amep_hvac_create_elbow`、`amep_hvac_create_damper`、`amep_hvac_create_fan` | 直管、弯头、风阀、风机 |
| 图层 | `amep_layer_list`、`amep_layer_set` | 图层列表与设置 |
| 系统/材料 | `amep_component_query`、`amep_gen_material_list`、`amep_gen_system_diagram` | 构件查询、材料表、系统图 |
| 绘图/标注 | `amep_draw_batch`、`amep_draw_entity`、`amep_plot_batch` | 批量绘图、实体绘制、批量出图 |

### `com_*` — COM 工具（53 个）

| 类别 | 代表工具 | 说明 |
|---|---|---|
| CAD 信息 | `com_get_cad_info` | 版本、文档、COM 版本 |
| 墙体 | `com_draw_wall` | COM 版直墙（简单参数） |
| 散热器 | `com_draw_radiator`、`com_draw_radiator_batch` | 散热器单/批量 |
| 绘图 | `com_draw_entity`、`com_draw_batch`、`com_draw_3d_solid`、`com_draw_sheet` | 通用实体、批量、三维实体、图签 |
| 标注 | `com_add_dimension`、`com_draw_dim_continue`、`com_dim_door` | 线性标注、连续标注、门洞标注 |
| 文字 | `com_create_text`、`com_create_mtext`、`com_edit_text_in_block` | 单行/多行文字、块内文字编辑 |
| 块 | `com_insert_block`、`com_insert_lib_block`、`com_manage_block`、`com_list_block_attributes` | 插入块、图库块、块管理、属性提取 |
| 图层 | `com_filter_objects`、`com_find_object`、`com_select_entities`、`com_filter_like` | 对象筛选/查找/选择 |
| 编辑 | `com_modify_entity`、`com_transform_entity`、`com_align_entities`、`com_parallel_align`、`com_change_block_color` | 修改/变换/对齐/平行对齐/改块色 |
| 表格 | `com_draw_sheet`、`com_edit_sheet_cell`、`com_export_sheet_to_excel`、`com_import_excel_to_sheet` | 表格绘制/编辑/Excel 互导 |
| 工具 | `com_manage_document`、`com_manage_style`、`com_manage_view`、`com_manage_xdata`、`com_get_variable`、`com_set_variable`、`com_send_command`、`com_zoom` | 文档/样式/视图/扩展数据/系统变量/命令行/缩放 |

---

## License

TBD — internal distribution; add `LICENSE` file before any external release.
