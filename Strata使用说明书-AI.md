# Strata 本地部署 - AI 可读操作手册

> 本文件面向 AI 编码助手。所有路径、命令、端点均为本机实测值。人类版说明书见同目录 `Strata使用说明书.pdf`，安装过程记录见 `Strata安装日志.md`。

## 0. 环境事实（本机）

| 键 | 值 |
| --- | --- |
| OS | Windows 10/11 x64（PowerShell 5.1，`&&` 不可用；复杂命令勿内联 `$` 变量，写 .ps1 执行） |
| GPU | NVIDIA RTX 5070 Ti, 16303 MiB VRAM, driver 616.56 |
| RAM | 95.7 GB |
| CPU | AMD Ryzen 7 9800X3D, 16 逻辑核, AVX2 支持 |
| 程序目录 | `D:\Strata`（git 仓库，main 分支） |
| 模型数据 | `D:\Strata-data`（85.8 GB，勿删勿移） |
| git | `D:\Program Files\Git\cmd\git.exe`（2.56.0，不在默认 PATH，需全路径或刷新环境） |
| Python venv | `D:\Strata\.venv\Scripts\python.exe`（Strata 专用，勿用于其他项目） |
| 网络 | github.com：git 可直连但 curl/SChannel TLS 失败(exit 35)；huggingface.co 不可达；modelscope.cn 可达；pypi 用清华镜像 `https://pypi.tuna.tsinghua.edu.cn/simple` |

## 1. 当前运行状态

- 模型：`qwen3.8-flash-next-iq3_s`（family=qwen, size=IQ3_S, 来源 ModelScope）
- 服务器：`http://127.0.0.1:8080`，仅本机监听，**无 API key**（`api_key: false`）
- 上下文：`max_context: 65536`；vision 关闭（`images: false`）；并发：1
- 实测速度：约 42.5 tokens/s（短回复、reasoning_effort=none）
- 健康检查返回样例：`{"status":"ok","max_context":65536,"model":"qwen3.8-flash-next-iq3_s","images":false,"api_key":false,"loaded":true,"service":"strata"}`

## 2. 启动 / 停止 / 状态检查

```powershell
# 启动（弹独立窗口；首次加载数分钟、期间系统可能卡顿，属正常；勿杀进程）
Start-Process -FilePath 'D:\Strata\run-iq3_s.bat' -WorkingDirectory 'D:\Strata'

# 就绪判定（轮询，后续启动 30-90s）
curl.exe -s http://127.0.0.1:8080/health   # "loaded": true 即就绪

# 停止：关闭服务器窗口。命令行方式（谨慎使用）：
# Get-CimInstance Win32_Process -Filter "Name like 'strata%'" | 终止对应进程
```

## 3. API 端点（全部已实测）

| 用途 | 端点 | 说明 |
| --- | --- | --- |
| 健康 | `GET http://127.0.0.1:8080/health` | 模型未加载时不响应 |
| 模型列表 | `GET http://127.0.0.1:8080/v1/models` | id 固定为 `qwen3.8-flash-next-iq3_s` |
| Chat | `POST http://127.0.0.1:8080/v1/chat/completions` | OpenAI 兼容；model 名可任意；响应含 `timings` 字段 |
| Anthropic | `POST http://127.0.0.1:8080/v1/messages` | Claude Code 等 |
| Responses | `POST http://127.0.0.1:8080/v1/responses` | Codex CLI（stateless） |

请求要点：
- `reasoning_effort`: `"none"|"low"|"medium"|"high"`（默认 high；none 最快）
- 有 API key 时加 `Authorization: Bearer <key>`；当前未设置 key
- 一次只处理一个请求，多客户端会排队

Claude Code 接入：`ANTHROPIC_BASE_URL=http://127.0.0.1:8080`，`ANTHROPIC_MODEL`=任意已知 Claude 名（被忽略），`ANTHROPIC_AUTH_TOKEN`=任意值。

## 4. 修改配置（重跑 setup，自动保留/续传）

```powershell
# 通用形式（--yes 免交互；改参数用 --setup）
& 'D:\Strata\.venv\Scripts\python.exe' 'D:\Strata\setup.py' <flags>

# 常用 flag：
#   --family qwen|swift|coder|unsloth   --model Q2_0|IQ2_XS|IQ3_XXS|IQ3_S|IQ1_M|UD-IQ4_XS|UD-Q4_K_XL
#   --context N        默认按 VRAM：<14GB→32768, <20GB→65536, 否则 131072（当前 16GB→65536）
#   --vision yes|no|gpu|cpu            当前 no
#   --parallel N       并发解码 N 路（每请求速度 -10~25%）
#   --port N           默认 8080；被占用时报错，改 8081
#   --vram-reserve-mib 1500   显卡驱动显示时防黑屏/冻结
#   --host H --api-key K      局域网暴露时两者必须同时给
#   --source modelscope       国内下载必加（huggingface.co 不可达）
#   --no-start         只装不启（agent 必用，否则前台阻塞）
#   --check            仅体检
#   --setup            修改已装模型的参数
```

换更小模型（内存吃紧时）：`--setup --model IQ2_XS` 或 `Q2_0`。

## 5. 故障规则（IF 症状 THEN 动作）

| IF | THEN |
| --- | --- |
| /health 无响应 | 检查服务器窗口是否开着；看 `D:\Strata\strata-iq3_s.log`；等 loaded=true |
| "port 8080 already in use" | 多半 Strata 已在运行（先查 /health）；否则 setup 加 `--port 8081` |
| 极慢 / 硬盘狂转 / "engine stopped unexpectedly" | 内存不足：关其他程序；或 `--setup --model IQ2_XS` |
| `ExpertCache: cudaMalloc failed: out of memory`（显存明明有余） | Windows 页面文件被禁用/过小：设"系统管理的大小"后重启 |
| "prompt exceeds the context" | `--setup --context <更大>` |
| NVIDIA driver too old | 让用户升级驱动（NVIDIA App / nvidia.com/drivers）并重启；**AI 不得自行装驱动** |
| 下载中断 | 重跑同一条 setup 命令（可续传，勿改参数） |
| 加载后黑屏/死机 | 显卡同时驱动显示：setup 加 `--vram-reserve-mib 1500` |

## 6. 安全红线（必须遵守）

1. 保持 `--host 127.0.0.1`。仅当用户明确要求局域网访问时才可 `--host 0.0.0.0`，且**必须**同时 `--api-key <长随机串>` 并把 key 告知用户。
2. 无 key 的服务器严禁置于隧道/端口转发之后。
3. 不要动 `D:\Strata-data`（85.8 GB）；不要把它移到 C 盘（C 盘仅剩约 80GB）。
4. 除 setup 自身行为外不要修改系统；装驱动、改页面文件前先征得用户同意。
5. 模型完全本地运行，无数据外传；勿引入任何外部 telemetry。

## 7. 文件索引

- 官方文档：`D:\Strata\docs\`（AI_SETUP.md 为 agent 安装总纲；TROUBLESHOOTING.md、DETAILS.md、MODELS.md、BATCHING.md 按需查）
- 安装日志：`D:\Strata\setup-run.log`；引擎日志：`D:\Strata\strata-iq3_s.log`
- 启动脚本：`D:\Strata\run-iq3_s.bat`（`START-HERE.bat` 等价，无 flag 时启动已装模型）
- 本手册与人类版：`C:\Users\CFFFY\Desktop\本地部署\`
