# 智行·端析 / Alpamayo 1.5：当前进度

更新时间：2026-09-17 19:44 CST。本文是**状态快照**；进行中的下载或构建须以完成后的退出码和验收记录更新，不能按“已完成”理解。

## 当前结论

官方源码、10B checkpoint 文件、一个冻结测试 clip 和独立离线兼容推理脚本已在本机准备；**尚未具备可确认的 L20 完全离线端到端推理结果**。最关键的未完成项是 Cosmos 非权重配置、Linux/amd64 的完整 uv 依赖环境、资产传输后复验与 L20 实机运行。没有进行剪枝、蒸馏、量化、ONNX 或 TensorRT 工作。

| 项目 | 当前状态 | 已有证据／边界 |
| --- | --- | --- |
| NVIDIA 官方源码 | 已核对 | `alpamayo1.5/` HEAD=`36aeb4c5938cbc2eb2aed33b22434773da4ab639`，工作树 clean；原始 `test_inference.py` SHA256=`dc69646feed09f92defa00a19ae6f2fc2a37a10a70ec35678f87946fbe6fe8e8`，未改动。 |
| Alpamayo 1.5 10B 权重 | 文件已准备 | 本机 `models/` 含五个 safetensors 分片、`config.json` 和索引；此前已按冻结大小／SHA256 校验，本次复查五片与两份 JSON 仍在。**尚未在 L20 加载**，传输后需重验。 |
| 一个测试 clip | 文件已验收 | `/Users/lang/Downloads/alp15-test-clip-delivery-20260917`：5 项 feature、4 项 metadata，冻结 `clip_id=030c760c-ae38-49aa-9ad8-f5650a545d26`、`t0_us=5100000`、`chunk_id=3119`；文件验收见 [ALP15-CLIP-ACCEPT-001.md](ALP15-CLIP-ACCEPT-001.md)。本机已在外联拦截下通过 `physical_ai_av==0.2.0` 读取 egomotion，**尚未完成四路视频与模型端到端测试**。 |
| 离线兼容推理代码 | 已实现、本机测试通过 | [脚本](../../scripts/test_alpamayo15_offline_compat.py)显式使用本地权重、clip、Cosmos/Qwen 配置，模型配置仅在内存中重映射，并阻止 Python socket 外联；项目 7 项本机测试通过。它不等于 NVIDIA 官方原始入口跑通，详见 [ALP15-OFFLINE-INFER-001.md](ALP15-OFFLINE-INFER-001.md)。 |
| Qwen 处理器配置 | 本机缓存可见 | 冻结 revision `89644892e4d85e24eaac8bacfd4f463576704203` 的九个非权重配置文件与 HF blob 身份已校验；还需作为完整缓存或带来源 manifest 的独立目录交付 L20。 |
| Cosmos 处理器／VLM 配置 | 缺失 | 冻结 revision `a9fae2cf89dc64db96b12860417f0eb403013bb9` 的九个非权重配置在项目目录及已检查本机缓存中未找到。**不需要 Cosmos 权重**。 |
| Linux/amd64 uv 环境 | 准备中，未验收 | 官方 `pyproject.toml` 要 Python 3.12、torch 2.8.0、transformers 4.57.1、physical-ai-av 0.2.0、flash-attn 2.8.3 等；锁文件有 flash-attn 源码包而无 wheel，构建需 CUDA 12.x `nvcc`。 |
| L20 官方／兼容推理 | 未执行 | L20 驱动、CUDA、依赖导入、官方入口阻塞记录、完整输出及资源观测均尚无实机验收证据。 |

## 正在进行的环境准备

本机 Docker Desktop 已启动。原有 `ubuntu:22.04` 实际是 `linux/arm64`，不能用来生成目标 L20 的 Linux/amd64 依赖。另有本地 `alpamayo:latest`（Ubuntu 22.04 / amd64 / CUDA 12.4.1），但它缺少 `nvcc`，其中 PyTorch 导入报 `libcusparseLt.so.0` 缺失，不能直接作为完整基线环境。

已从 NVIDIA 官方 Docker Hub 核对 `nvidia/cuda:12.8.1-devel-ubuntu22.04` 提供 `linux/amd64` 版本；截至本文时间，正在执行 `docker pull --platform linux/amd64 nvidia/cuda:12.8.1-devel-ubuntu22.04`，**尚无完成退出码**。拉取完成后仍需确认镜像摘要、`nvcc`、Python 3.12 与 uv，再按冻结 `uv.lock` 安装并用全新环境做 `uv sync --frozen --offline`。不将 Mac 或容器内的 `.venv` 直接视为可搬运的 L20 交付物；目标交付是经验证的 Linux/amd64 uv 缓存／构建产物，并在 L20 断网重建环境。

## 下一步与验收门槛

1. 完成 amd64 CUDA 开发镜像拉取、确认摘要和工具链；构建完整锁定依赖，尤其验证 `flash-attn`，保留命令、日志、退出码、镜像摘要与 wheel 哈希。若编译或离线重建失败，保留失败证据，不宣称环境完成。
2. 在有网准备机补齐 Cosmos 的**九个非权重文件**；Qwen 配置一起按冻结 revision 校验与封装。不要重下已验收的 10B 权重或 clip，也不要下载 Cosmos/Qwen 权重。
3. 向 L20 传输源码、模型、clip、配置及 Linux/amd64 uv 缓存；在 L20 重新验证文件哈希、架构、驱动/CUDA、Python 和关键依赖。
4. 按 [AGENTS.md](../../AGENTS.md) 先记录 NVIDIA 官方原始入口在断网环境下的实际结果；若确有已记录的阻塞，再运行已批准的离线兼容入口。保存完整日志、退出码、输出形状、CoC、minADE、内存／显存与耗时，由 Sol 复核。**兼容入口成功不能命名为官方原始入口成功**。

本文件不包含 token、密码、SSH 地址、GPU UUID 或带签名下载 URL；完整过程输出以当前任务工具调用记录及后续单独的构建日志为准。
