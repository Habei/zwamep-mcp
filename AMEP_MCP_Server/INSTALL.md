# ZwAmepMcpServer 安装说明

ZwZwAmepMcpServer.exe 是中望 AMEP（建筑水暖电）MCP Server 的单文件发行版（PyInstaller 打包，目标机无需安装 Python）。

它把 AI 客户端（Claude 等 MCP 客户端）与 ZWCAD/AMEP 连起来，提供 110 个工具：57 个命名管道工具（`amep_*` 前缀，建筑构件/风管/风阀/风机等）+ 53 个 COM 工具（`com_` 前缀，通用绘图/编辑/标注等）。

**重要：exe 只是"前端"。** 真正的画图能力在 ZWCAD 进程内的两个 ZRX(ZAecAMEPServerBridge.zrx、ZAecAMEPServerCom.zrx) 插件里，二者必须按下文安装，否则 exe 能启动但所有 CAD 工具都报错。

## 系统要求

| 项 | 要求 |
|---|---|
| 操作系统 | Windows 10/11 x64 |
| CAD | ZWCAD 2027（暖通版，即 ZWHVAC 2027） |
| MCP 客户端 | 任意支持 stdio 传输的客户端（Claude Desktop / Claude Code / Cursor 等） |
| Python | **不需要**（已打包进 exe） |


## 安装步骤

### 1. 放置文件

把分发包解压到任意固定目录，例如 `C:\AMEP_MCP\`。

### 2. ZWCAD 加载 ZRX 插件

启动 ZWCAD 2027，命令行执行 `APPLOAD`：

- 加载 `zrx\ZAecAMEPServerBridge.zrx` → 命令行出现 `[ZAecAMEPServerBridge] 桥接插件已加载，命名管道就绪。MCP Server 可连接。` 即成功。
- 加载 `zrx\ZAecAMEPServerCom.zrx`（COM 工具需要）。

验证：
- 命令行执行 `AMEPSERVERBRIDGE`，应输出 `命名管道服务: 运行中 (管道名: \\.\pipe\amep_server_bridge_pipe)`。

**开机自动加载（推荐）**：`APPLOAD` 对话框底部「启动套件（Startup Suite）」→ 添加上述两个 zrx，下次启动 ZWCAD 自动加载。

### 3. 配置 MCP 客户端

在 MCP 客户端配置中加入（以 Claude Desktop 的 `claude_desktop_config.json` 为例）：

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

Claude Code 用户等价命令：

```
claude mcp add amep -- "C:\AMEP_MCP\ZwAmepMcpServer.exe" --mode named_pipe
```

> exe 路径按实际放置位置修改。`--mode named_pipe` 可省略（默认即 named_pipe），显式写出便于排障。

### 4. 验证

1. 启动 ZWCAD 并新建/打开图纸（如 Drawing1.dwg）。
2. 在 MCP 客户端里调用 `amep_get_app_info` → 应返回 ZWCAD 版本与当前文档。
3. 调用 `com_get_cad_info` → 应返回 `cad_name: ZWHVAC`、`com_version: ZAecAMEPServerCom 1.0`。
4. 画个东西试试：`com_draw_radiator`（x=0, y=0）→ ZWCAD 里出现散热器即全链路通。

## 可选配置（config\amep_mcp_config.json）

exe 启动时优先读 exe 同目录 `config\amep_mcp_config.json`，缺省使用内置默认。可调项：

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

## 故障排查

| 现象 | 原因与处理 |
|---|---|
| MCP 客户端连不上 exe | command 路径写错；或被杀软/SmartScreen 拦截（exe 未签名），加白名单后重试 |
| `amep_*` 工具全部超时/报 pipe 错误 | ZWCAD 没加载 `ZAecAMEPServerBridge.zrx`，或加载的不是同版本编译产物；ZWCAD 里执行 `AMEPSERVERBRIDGE` 确认管道运行中 |
| `com_*` 工具报 "该 ProgID 由 ZAecAMEPServerCom.zrx 加载" 相关错误 | ZWCAD 没加载 `ZAecAMEPServerCom.zrx` |
| `com_*` 工具可用但 `amep_*` 报 loadModule 失败 | `ZAecArchService.zrx`/`ZAecHvacService.zrx` 及其依赖 dll 不在搜索路径；整目录拷贝 |
| 管道工具调了但图纸无变化 | 确认 ZWCAD 有打开的图纸；部分工具需先有宿主实体（如门窗需先有墙） |
| 加载 zrx 报缺 dll | 依赖 dll 未随包分发；从开发机 `Out\VC15\Alpha_HVAC\x64\Bin\HVAC27\` 整目录拷贝 |
| 版本不匹配/加载后无反应 | zrx 与 ZWCAD 版本必须配套（当前为 2027/HVAC27），跨版本不通用 |

## 注意事项

- 单 ZWCAD 实例：COM 连接绑定到正在运行的 ZWCAD 进程，多开时连的是先启动（已注册 ProgID）的那个，建议只开一个。
- 编译更新：若 zrx 重新编译，需在 ZWCAD 里卸载旧版再加载新版（或重启 ZWCAD），管道断开后 MCP 工具会自动重连。
- exe 与 zrx 配套分发：exe 内置的协议与 zrx 的管道协议/方法名是配套版本，请整包一起升级，不要单独替换其一。
