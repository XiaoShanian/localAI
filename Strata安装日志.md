# Strata 本地部署安装日志

- **日期**：2026-10-10
- **目标**：在本机部署 [Strata](https://github.com/Niko1221/Strata)，按照仓库 `docs/AI_SETUP.md` 流程执行
- **结果**：✅ 部署成功，服务器运行于 `http://127.0.0.1:8080`

---

## 一、本机环境（部署前检查）

| 项目 | 配置 | 是否满足要求 |
| --- | --- | --- |
| GPU | NVIDIA GeForce RTX 5070 Ti，16GB VRAM | ✅（要求 12GB+，RTX 50 系支持） |
| 显卡驱动 | 616.56 | ✅（要求 NVIDIA 580+） |
| 内存 | 95.7 GB | ✅（要求 32GB+；96GB 档推荐 IQ3_S） |
| CPU | AMD Ryzen 7 9800X3D（8 核 16 线程，支持 AVX2） | ✅ |
| 磁盘 | C: 剩余 80GB / D: 剩余 452GB | ✅（模型约 70-86GB，放 D 盘） |
| 操作系统 | Windows 10/11 x64 | ✅ |

## 二、依赖准备

1. **git**：本机原本未安装；由用户手动安装到 `D:\Program Files\Git`（git 2.56.0.windows.2）。
2. **网络连通性测试**：
   - `https://github.com`：curl（SChannel）TLS 握手失败（exit 35），但 **git 自带 OpenSSL 可直连**，克隆无障碍；
   - `https://huggingface.co`：❌ 不可达；
   - `https://modelscope.cn`：✅ 可达 → 模型下载改用 ModelScope 源。
3. **Python**：由 Strata 的 `START-HERE.bat` 自动安装（用户级，无需管理员），并在 `D:\Strata\.venv` 创建私有虚拟环境。

## 三、安装步骤

| 步骤 | 操作 | 结果 |
| --- | --- | --- |
| 1 | `git clone https://github.com/Niko1221/Strata.git D:\Strata` | ✅ 克隆成功（main 分支） |
| 2 | 阅读 `docs/AI_SETUP.md`，按流程逐项执行 | ✅ |
| 3 | 硬件检查（nvidia-smi / CIM / 磁盘） | ✅ 全部达标 |
| 4 | 与用户确认：模型选 **Qwen IQ3_S**（约 70GB），同意长时间下载 | ✅ |
| 5 | 后台运行 `START-HERE.bat --yes --family qwen --model IQ3_S --no-start --source modelscope` | ✅ 日志：`D:\Strata\setup-run.log` |
| 6 | setup 自动完成：Python 安装 → .venv → pip 依赖 → 下载预编译 CUDA 引擎（sm_120，SHA-256 校验通过）→ 安装 cuBLAS/CUDA 运行时 → 从 ModelScope 下载模型 → 生成启动脚本 `run-iq3_s.bat` | ✅ |
| 7 | 模型落盘检查：`D:\Strata-data` 共 **85.8 GB**（GGUF 分片 51.05 + 26.82 GB，另含 MTP 草稿层与 packs） | ✅ |
| 8 | 启动服务器：`Start-Process D:\Strata\run-iq3_s.bat` | ✅ 独立窗口运行 |
| 9 | 验证 `/health`：`{"status":"ok","model":"qwen3.8-flash-next-iq3_s","loaded":true,"max_context":65536}` | ✅ |
| 10 | 验证 `/v1/models` 与 `/v1/chat/completions` 真实推理 | ✅ 回复正常，约 **42.5 tokens/s** |

## 四、遇到的问题与解决

| 问题 | 原因 | 解决 |
| --- | --- | --- |
| curl 访问 GitHub 失败（exit 35） | Windows curl 使用 SChannel，TLS 握手被干扰 | 改用 git（自带 OpenSSL）直连，成功 |
| huggingface.co 不可达 | 国内网络限制 | setup 原生支持 ModelScope：加 `--source modelscope` |
| setup 后台任务报 exit 1 | bat 脚本退出码怪癖，日志末尾实为 `All set.` | 确认安装已完成，无需处理 |
| PowerShell 内联命令 `$` 变量被吞 | 外层 shell 转义问题 | 复杂命令写成 .ps1 脚本执行 |

## 五、部署产物位置

- 程序目录：`D:\Strata`
- 模型数据：`D:\Strata-data`（85.8 GB）
- 启动脚本：`D:\Strata\run-iq3_s.bat`（或 `START-HERE.bat`）
- 安装日志：`D:\Strata\setup-run.log`
- 引擎运行日志：`D:\Strata\strata-iq3_s.log`
- 本说明文档：`C:\Users\CFFFY\Desktop\本地部署\`

## 六、后续维护要点

- 模型下载可断点续传：若中断，重跑第 5 步同一条命令即可继续。
- 更换/调整模型：`START-HERE.bat --setup`（如改量化大小、上下文、并发）。
- 若加载后黑屏/卡顿（显卡同时驱动显示）：重跑 setup 加 `--vram-reserve-mib 1500`。
- 需要多客户端并发：配置 `--parallel N`（该卡上每请求速度损失约 10-25%）。
